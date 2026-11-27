# Sticky Bit — Restricted Deletion on Shared Directories

[← SGID](../02-sgid/README.md) · [Main guide](../README.md)

**Special octal digit:** `1` | **Symbol:** `t` / `T` in others execute position | **Typical mode:** `1777`

## 1. The kid-friendly analogy: the classroom noticeboard

Imagine a noticeboard that all students can use. Alice pins up her drawing, Bob pins up his homework, and everyone can add their own work. Without a special rule, any student could remove another student's work.

The teacher introduces a rule: students may remove their own work, but not another student's work. The teacher can still remove any work from the noticeboard.

| Classroom story | Linux equivalent |
| --- | --- |
| Noticeboard | Shared writable directory |
| Student | Linux user |
| Drawing or homework | Directory entry, e.g. a file |
| Teacher | Directory owner or appropriately privileged process |
| New removal rule | Sticky Bit |

**Core idea:** The Sticky Bit restricts deletion and renaming of entries in a directory. A user must normally be the entry's owner, the directory owner, or a suitably privileged process, and must satisfy the other applicable access checks.

## 2. Real-world Linux scenario: `/tmp`

Applications, services and users share `/tmp` for temporary files. They must be able to create files without being able to casually delete files belonging to others.

Inspect your system without changing anything:

```bash
ls -ld /tmp /var/tmp /dev/shm
stat -c '%a %A %U:%G %n' /tmp
```

Typical result for `/tmp`:

```text
drwxrwxrwt  ... root root ... /tmp
1777 drwxrwxrwt root:root /tmp
```

`1777` gives normal read, write and traverse permissions to everyone, but adds the Sticky Bit restriction. Compare with `0777`:

| Operation by an ordinary user who does not own the directory | `0777` | `1777` |
| --- | --- | --- |
| Create an entry | Allowed | Allowed |
| Remove or rename own entry | Allowed | Allowed |
| Remove or rename another user's entry | Normally allowed | Normally denied |
| Read file contents | Depends on file access | Depends on file access |
| Modify file contents | Depends on file access | Depends on file access |

The table assumes a writable filesystem, accessible parent paths, and no other security restrictions. Replacing an existing destination during a rename can require an additional ownership check on that destination.

### Why can someone delete a read-only file?

Imagine `report.txt` has mode `0444` (`-r--r--r--`). No ordinary write bit is set. But if the file is inside an accessible directory with mode `0777`, another user can normally delete it.

```text
Edit the contents                    Delete the name
       |                                    |
       v                                    v
Check file write access            Check parent directory write + search
and path traversal                 plus Sticky Bit ownership (if set)
```

Deletion (`unlink`) removes a name from its parent directory. It does not require the file's ordinary write bit. File data may remain until all hard links and open references are gone. Other checks, such as immutable attributes or mandatory security policy, may prevent deletion.

Sticky Bit does **not** grant new access, make file contents read-only, encrypt files, or prevent authorized users from editing them. It adds an ownership-based restriction to directory entry removal and renaming.

Other typical uses include `/var/tmp`, `/dev/shm` and carefully designed shared upload or artifact directories. Actual installed modes vary.

## 3. Hands-on WSL lab: Alice vs. Bob

**Goal:** Demonstrate that a read-only file can be deleted in a `0777` directory, then prove that Sticky Bit prevents the same operation in a `1777` directory.

This lab creates two temporary password-locked accounts with no home directories, using the existing unprivileged group of `nobody`. It does not modify the real `/tmp` permissions. Run all steps in one Bash terminal, preferably as a regular user with sudo access. The helper also works from a root shell.

<!-- lab-start -->
### Step 1 — Create an isolated playground

```bash
as_root() { if [ "$(id -u)" -eq 0 ]; then "$@"; else sudo "$@"; fi; }
LAB_DIR=$(mktemp -d /tmp/sticky-lab.XXXXXX)
chmod 0755 "$LAB_DIR"
cd "$LAB_DIR"
pwd
findmnt -T . -o TARGET,FSTYPE
```

Stop if setup fails or the filesystem is a Windows mount. The outer directory is traversable so the temporary users can reach the test directory. Only the inner test directory will be world-writable.

### Step 2 — Create two unprivileged test users

```bash
ALICE="sticky_a_$$"
BOB="sticky_b_$$"
TEST_GROUP=$(id -gn nobody)
NOLOGIN=/sbin/nologin
[ -x /usr/sbin/nologin ] && NOLOGIN=/usr/sbin/nologin
as_root useradd -M -N -g "$TEST_GROUP" -s "$NOLOGIN" "$ALICE"
as_root useradd -M -N -g "$TEST_GROUP" -s "$NOLOGIN" "$BOB"
id "$ALICE"
id "$BOB"
```

Continue only if both accounts were created successfully. Do not reuse existing accounts; if the second creation fails, remove only the first account if it was successfully created by this lab. The accounts have no assigned passwords or login shell access. `runuser` will launch specific commands as them without an interactive login.

| Option | Purpose |
| --- | --- |
| `-M` | Do not create a home directory |
| `-N` | Do not create a private group |
| `-g` | Use the specified existing primary group |
| `-s` | Set a non-login shell |
| `$$` | Current Bash process ID, to distinguish lab accounts |

### Step 3 — Make a root-owned shared directory WITHOUT Sticky Bit

```bash
SHARED="$LAB_DIR/shared"
as_root mkdir "$SHARED"
as_root chmod 0777 "$SHARED"
ls -ld "$SHARED"
```

Expected:

```text
drwxrwxrwx 2 root root ... shared
```

The directory is root-owned, so neither temporary user qualifies as its owner. The world-writable mode is intentional and temporary, not a production recommendation.

### Step 4 — Create two files, one read-only

```bash
as_root runuser -u "$ALICE" -- touch "$SHARED/alice.txt"
as_root runuser -u "$BOB" -- touch "$SHARED/bob.txt"
as_root runuser -u "$ALICE" -- chmod 0444 "$SHARED/alice.txt"
ls -l "$SHARED"
```

Example (Ubuntu group names and sample PIDs):

```text
-r--r--r-- 1 sticky_a_1234 nogroup ... alice.txt
-rw-r--r-- 1 sticky_b_1234 nogroup ... bob.txt
```

The mode on Bob's file can differ with the account's umask. Alice's file is explicitly read-only.

### Step 5 — Bob attempts to delete Alice's read-only file

Predict the result before running the command.

```bash
as_root runuser -u "$BOB" -- rm -f -- "$SHARED/alice.txt"
ls -l "$SHARED"
```

Expected: only `bob.txt` remains. Bob is allowed to remove the directory entry because he has parent-directory write and traverse permissions. `rm -f` avoids confirmation prompts; it does not bypass the kernel's permission checks.

### Step 6 — Recreate Alice's file and enable Sticky Bit

```bash
as_root runuser -u "$ALICE" -- touch "$SHARED/alice.txt"
as_root runuser -u "$ALICE" -- chmod 0444 "$SHARED/alice.txt"
as_root chmod +t "$SHARED"
ls -ld "$SHARED"
stat -c '%a %A %n' "$SHARED"
```

Expected:

```text
drwxrwxrwt 2 root root ... shared
1777 drwxrwxrwt .../shared
```

The equivalent complete mode command is `chmod 1777 "$SHARED"` when run by the directory owner or administrator. The final `x` has become `t`.

### Step 7 — Bob repeats the same deletion and tries renaming

```bash
as_root runuser -u "$BOB" -- rm -f -- "$SHARED/alice.txt"
as_root runuser -u "$BOB" -- mv -- "$SHARED/alice.txt" "$SHARED/alice-renamed.txt"
ls -l "$SHARED"
```

Expected intentional errors (wording can differ):

```text
rm: cannot remove '.../alice.txt': Operation not permitted
mv: cannot move '.../alice.txt' to '.../alice-renamed.txt': Operation not permitted
```

Both files should remain. Bob has normal directory write and traverse rights, but cannot pass the Sticky Bit ownership restriction.

### Step 8 — Alice renames and deletes her own file

```bash
as_root runuser -u "$ALICE" -- mv -- "$SHARED/alice.txt" "$SHARED/alice-renamed.txt"
ls -l "$SHARED"
as_root runuser -u "$ALICE" -- rm -f -- "$SHARED/alice-renamed.txt"
ls -l "$SHARED"
```

Expected: Alice successfully renames and removes her own entry. Bob's file remains.

### Step 9 — Root removes Bob's file

```bash
as_root rm -- "$SHARED/bob.txt"
ls -l "$SHARED"
```

Expected: `total 0`. Root owns this shared directory and normally also has the privileges to override the ownership restriction.

The three ownership/privilege exceptions are the entry owner, directory owner, and a privileged process (on Linux, the relevant capability is `CAP_FOWNER`). Ordinary path, directory and filesystem access checks still apply.

### Step 10 — Understand `t` vs. `T`

```bash
as_root chmod 1777 "$SHARED"
ls -ld "$SHARED"
as_root chmod 1776 "$SHARED"
ls -ld "$SHARED"
as_root chmod 1770 "$SHARED"
ls -ld "$SHARED"
```

Expected modes:

```text
drwxrwxrwt   1777
drwxrwxrwT   1776
drwxrwx--T   1770
```

| Symbol at others execute position | Sticky Bit | Others execute/traverse |
| --- | --- | --- |
| `x` | Off | On |
| `-` | Off | Off |
| `t` | On | On |
| `T` | On | Off |

Uppercase `T` is not a broken Sticky Bit. For example, `1770` can be intentional for a team-only directory. Users in the group can traverse it while outsiders cannot. In `1776`, outsiders have a write bit but lack the traversal needed for ordinary file creation or deletion.

### Step 11 — Remove Sticky Bit and verify

```bash
as_root chmod 1777 "$SHARED"
as_root chmod -t "$SHARED"
ls -ld "$SHARED"
```

Expected: `drwxrwxrwx`. The numeric equivalent for this specific starting mode is `0777`.

### Step 12 — Clean up the playground and test accounts

```bash
as_root chmod 1777 "$SHARED"
as_root rm -f -- "$SHARED/alice.txt" "$SHARED/alice-renamed.txt" "$SHARED/bob.txt"
as_root rmdir -- "$SHARED"
cd /tmp
rmdir -- "$LAB_DIR"
as_root userdel "$ALICE"
as_root userdel "$BOB"
```

Only the explicitly named test files, directories and accounts are removed. If cleanup fails, inspect the remaining entries rather than using an unrestricted recursive delete. Do not remove the existing `nobody` group.
<!-- lab-end -->

## 4. WSL gotchas and troubleshooting

- Use native Linux storage under WSL 2, e.g. `/tmp`, not `/mnt/c` or `/mnt/d`. Windows-mounted drives may not enforce the same permissions as a native Linux filesystem, even if mode bits are visible.
- Confirm the test directory filesystem using `findmnt -T . -o TARGET,FSTYPE,OPTIONS`. The `nosuid` mount option affects set-ID execution, not Sticky Bit deletion protection itself.
- If the unauthorized deletion succeeds with Sticky Bit enabled, check `ls -ld` for directory ownership and mode, `ls -l` for entry ownership, and `runuser -u ... -- id` to confirm the acting user is not root. Verify you are testing the intended native Linux directory.
- If file modification is denied, remember that Sticky Bit is not the only security mechanism. Linux hardening settings (such as `fs.protected_regular` in sticky writable directories), access bits, ACLs and other policies may affect file opening. Do not disable these settings for a lab.
- On modern Linux, the Sticky Bit has no standard useful permission effect on regular files. Historically, some Unix systems used it for executable text caching. Focus on directory protection.

## 5. Admin command reference and security

| Task | Command |
| --- | --- |
| Enable Sticky Bit without changing other bits | `chmod +t directory` |
| Set a public shared directory mode | `chmod 1777 directory` |
| Set a private group directory mode | `chmod 1770 directory` |
| Remove Sticky Bit | `chmod -t directory` |
| Inspect the directory itself | `ls -ld directory` |
| Find sticky directories | `sudo find /tmp /var/tmp /dev/shm -type d -perm -1000` |
| Combine SGID and Sticky Bit | `chmod 3770 directory` |

Never remove the Sticky Bit from the real `/tmp` merely to experiment. Avoid broad `777` permissions on production directories. Use a private group, carefully selected access permissions, and an explicit deletion policy as appropriate.

## 6. Interview questions and answers

**What is Sticky Bit?** A directory special permission that restricts deletion and renaming to the entry owner, directory owner, or appropriately privileged process, subject to other access checks.

**Why does `/tmp` have `1777` rather than `777`?** Users need to create temporary files, but ordinary users should not be able to remove or rename other users' entries.

**Can you delete a `444` file?** Yes, if the parent directory permissions and applicable ownership/security checks allow it. File-content write permission is not required for unlinking a name.

**Can Sticky Bit prevent file editing?** Not by itself. Modification is controlled by separate file access rules and security policies.

**What does uppercase `T` mean?** The Sticky Bit is set, but the others traverse/execute bit is absent.

**Can root delete another user's file inside a sticky directory?** Normally yes, when the process has the appropriate privileges and other constraints permit it. The directory owner can also pass the ownership restriction.

**Can SGID and Sticky Bit be combined?** Yes. `3770` on a team directory enables both group inheritance and ownership-based removal protection while limiting normal access to owner/group.

## References

- [inode(7)](https://man7.org/linux/man-pages/man7/inode.7.html)
- [unlink(2)](https://man7.org/linux/man-pages/man2/unlink.2.html)
- [rename(2)](https://man7.org/linux/man-pages/man2/rename.2.html)
- [chmod(1)](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [Microsoft Learn — WSL file permissions](https://learn.microsoft.com/en-us/windows/wsl/file-permissions)

---

**Remember:** Sticky Bit answers *who may delete or rename an entry in this shared directory?* It does not control the contents of that entry.
