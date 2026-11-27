# SGID — Set Group ID

[← SUID](../01-suid/README.md) · [Main guide](../README.md) · [Next: Sticky Bit →](../03-sticky-bit/README.md)

**Special octal digit:** `2` | **Symbol:** `s` / `S` in group execute position | **Typical modes:** `2755` executable, `2770` directory

## 1. The kid-friendly analogy: the science club

Imagine you belong to the Sports Club, but your teacher allows you to work in the Science Club's room. Every new project created there automatically receives the Science Club's label, regardless of the creator's own club. You remain the project's owner. Science Club members can edit the project if their permissions allow it.

That is SGID on a **directory**. On an **executable**, imagine a trusted machine temporarily using the Science Club's access badge while carrying out a particular task. You do not become a permanent member of that club.

| SGID applied to | Effect on Linux |
| --- | --- |
| Directory | New files inherit the parent directory's group; new subdirectories also inherit the SGID bit |
| Supported executable with group execute set | Program starts with the file's group as its effective GID (subject to security restrictions) |

## 2. Real-world Linux scenario: shared DevOps project

Three engineers work in a directory owned by the group `devops`.

| User | Usual primary group | Shared team |
| --- | --- | --- |
| Rahul | `rahul` | `devops` |
| Amit | `amit` | `devops` |
| Priya | `priya` | `devops` |

Without SGID, files normally take the effective group of the creating process (on typical Linux filesystem configurations). This leads to inconsistent group ownership. An administrator can configure an existing shared directory with an existing `devops` group like this:

```bash
# Illustrative production configuration, NOT a lab command.
sudo chgrp devops /projects/devops
sudo chmod 2770 /projects/devops
```

Assuming the directory is appropriately owned and all team members have group access:

```text
Without SGID                     With SGID
projects/devops/                 projects/devops/ (2770)
├── main.tf     rahul:rahul      ├── main.tf     rahul:devops
├── Dockerfile  amit:amit        ├── Dockerfile  amit:devops
└── deploy.sh   priya:priya      └── deploy.sh   priya:devops
```

Notice that the individual owners are preserved, while the group is consistent. Common use cases include team repositories, Terraform configuration directories, shared build artifacts and Ansible playbooks.

**SGID does NOT automatically make files group-writable.** Ordinary permissions, the creation mode, umask and default ACLs determine the initial access bits. Existing files are not retroactively changed. Moving an existing file into the directory with a same-filesystem rename generally preserves its existing group and mode; that is not creation of a new inode.

### SGID on executable programs

Some distributions install programs such as `wall` or `write` with SGID to use the `tty` group's access to terminals. Their installation modes can differ or the programs may not be installed.

```bash
ls -l /usr/bin/wall /usr/bin/write
```

Illustrative example:

```text
-rwxr-sr-x 1 root tty ... /usr/bin/wall
```

The program can start with the file's effective group ID (EGID), while its real GID and user identity remain unchanged by the SGID bit itself. The executable is responsible for using that access safely.

## 3. Hands-on WSL lab

Two experiments share a temporary, unprivileged group that the caller does not join. No existing group memberships are changed. Run in one Bash session, preferably as a normal user with sudo. The helper also supports a root shell; your baseline UID/GID will differ in that case.

<!-- lab-start -->
### Experiment A — Group inheritance on directories

#### Step 1 — Create a playground and temporary group

```bash
as_root() { if [ "$(id -u)" -eq 0 ]; then "$@"; else sudo "$@"; fi; }
mkdir -p "$HOME/linux-permissions-lab"
LAB_DIR=$(mktemp -d "$HOME/linux-permissions-lab/sgid.XXXXXX")
cd "$LAB_DIR"
LAB_GROUP="sgidlab_$$"
as_root groupadd "$LAB_GROUP"
getent group "$LAB_GROUP"
id
findmnt -T . -o TARGET,FSTYPE
```

Stop if setup or group creation fails; do not reuse an existing group. `$$` gives the Bash process ID and helps distinguish lab groups. The temporary group is deliberately not added to the user's memberships.

#### Step 2 — Create two identical directories without SGID

```bash
mkdir normal shared
as_root chgrp "$LAB_GROUP" normal shared
as_root chmod g-s normal shared
as_root chmod 0770 normal shared
ls -ld normal shared
(umask 0022; touch normal/before.txt shared/before.txt)
ls -l normal shared
```

Example (Rahul, temporary group `sgidlab_1234`):

```text
drwxrwx--- 2 rahul sgidlab_1234 ... normal
drwxrwx--- 2 rahul sgidlab_1234 ... shared
normal/before.txt:  -rw-r--r-- rahul rahul
shared/before.txt:  -rw-r--r-- rahul rahul
```

The lines showing files are simplified for readability. The user owns both directories and has permission to create entries even without membership in the temporary group. Without SGID, the new files normally use the caller's effective group.

#### Step 3 — Enable SGID on only one directory

```bash
as_root chmod g+s shared
ls -ld normal shared
(umask 0022; touch normal/after.txt shared/after.txt)
ls -l normal shared
```

Example:

```text
normal/                   shared/
0770 drwxrwx---            2770 drwxrws---

before.txt  rahul:rahul    before.txt  rahul:rahul
after.txt   rahul:rahul    after.txt   rahul:sgidlab_1234
```

The two existing `before.txt` files are unchanged. Only new entries created inside `shared` get its group. The equivalent complete numeric mode is `chmod 2770 shared` (as administrator here).

#### Step 4 — Test subdirectory inheritance

```bash
(umask 0022; mkdir -p shared/child/grandchild; touch shared/child/grandchild/test.txt)
ls -ld shared/child shared/child/grandchild
ls -l shared/child/grandchild
```

Expected modes and group (without inherited default ACLs):

```text
drwxr-sr-x ... sgidlab_1234 ... shared/child
drwxr-sr-x ... sgidlab_1234 ... shared/child/grandchild
-rw-r--r-- ... sgidlab_1234 ... test.txt
```

New directories inherit both the group and SGID. New regular files inherit group ownership, but SGID is not automatically set on those regular files. The subdirectories are `2755` here because of the controlled `0022` umask; `2770` on the parent does not force identical access bits on its children.

#### Step 5 — Observe SGID and umask together

```bash
umask
(umask 0002; touch shared/teamwork.txt)
(umask 0007; touch shared/private-team.txt)
ls -l shared/teamwork.txt shared/private-team.txt
```

Example, assuming a normal `0666` requested file creation mode and no default ACL:

| Creation umask | Typical file mode | Group write? | Others read? |
| --- | --- | --- | --- |
| `0022` | `0644` | No | Yes |
| `0002` | `0664` | Yes | Yes |
| `0007` | `0660` | Yes | No |

All new files in `shared` still belong to the same inherited group. Parent traversal restrictions can prevent outsiders from accessing a file even if its own mode permits reading. Parentheses keep the temporary umask changes inside a subshell.

### Experiment B — Effective group identity on an executable

#### Step 6 — Make a copy of `id` and change its group

```bash
cp /usr/bin/id "$LAB_DIR/demo-id"
chmod 0755 "$LAB_DIR/demo-id"
as_root chgrp "$LAB_GROUP" "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

Expected: the file belongs to the temporary group, but the process still has the caller's group identity.

#### Step 7 — Enable SGID and compare real/effective GIDs

```bash
as_root chmod g+s "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
"$LAB_DIR/demo-id" -rg
"$LAB_DIR/demo-id" -g
id -gn
```

Example:

```text
-rwxr-sr-x 1 rahul sgidlab_1234 ... demo-id
uid=1000(rahul) gid=1000(rahul) egid=1002(sgidlab_1234) groups=...
1000
1002
rahul
```

`id -rg` reports real GID; `id -g` reports effective GID. The caller's shell and permanent group memberships did not change. The kernel applies the executable's group as its effective GID when the set-ID conditions are met.

#### Step 8 — Understand `s` vs. `S`

```bash
as_root chmod 2755 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
as_root chmod 2644 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

Expected modes and intentional error:

```text
-rwxr-sr-x
-rw-r-Sr--
bash: .../demo-id: Permission denied
```

| Symbol at group execute position | SGID bit | Group execute bit |
| --- | --- | --- |
| `x` | Off | On |
| `-` | Off | Off |
| `s` | On | On |
| `S` | On | Off |

The mode `2644` has no execute bits, so direct execution fails. In a different mode with uppercase `S`, the owner or others may still have execute permission, but the Linux executable set-GID identity change requires group execute to be set. On a directory, uppercase `S` does not disable group inheritance; it only indicates missing group traversal.

#### Step 9 — Remove SGID and verify

```bash
as_root chmod g-s "$LAB_DIR/demo-id"
as_root chmod 0755 "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
as_root chmod g-s shared
ls -ld shared shared/child
```

SGID is gone on `shared`, but existing child directories retain their own inherited SGID bit. GNU `chmod` can preserve unmentioned directory set-ID bits with ordinary numeric modes; use `chmod g-s` to explicitly remove SGID.

#### Step 10 — Clean up

```bash
as_root chmod g-s "$LAB_DIR/demo-id"
rm -- "$LAB_DIR/demo-id"
rm -- normal/before.txt normal/after.txt shared/before.txt shared/after.txt
rm -- shared/teamwork.txt shared/private-team.txt shared/child/grandchild/test.txt
rmdir shared/child/grandchild shared/child normal shared
cd "$HOME/linux-permissions-lab"
rmdir -- "$LAB_DIR"
as_root groupdel "$LAB_GROUP"
```
<!-- lab-end -->

## 4. WSL gotchas and troubleshooting

- Practice on the native Linux filesystem in WSL 2; do not assume Windows-mounted `/mnt/c` handles group ownership or special permission bits in the same way.
- Check `findmnt -T . -o TARGET,FSTYPE,OPTIONS` inside the lab. `nosuid` and `no_new_privs` can suppress executable set-GID credential changes; they do not by themselves disable directory group inheritance.
- Filesystems mounted with group-inheritance semantics may inherit the parent group even without SGID. The comparison assumes standard Linux creation behavior on the native filesystem.
- If the SGID bit on a regular file disappears, check who owns the file and whether the person setting it belongs to its group; an unprivileged caller may be unable to retain SGID on a file in a different group. Ownership changes and writes can also clear set-ID bits.
- Local defaults such as default ACLs can change file permission results. Inspect with `getfacl` if installed. Avoid changing global security settings for the lab.

## 5. Admin reference and security

| Task | Command |
| --- | --- |
| Enable SGID | `chmod g+s path` |
| Enable SGID with private team access | `chmod 2770 directory` |
| Enable SGID on a supported binary | `chmod 2755 file` |
| Remove SGID explicitly | `chmod g-s path` |
| Change group ownership | `chgrp group path` |
| Inspect effective group | `id -g` / `id -gn` |
| Find SGID regular files | `sudo find /usr -type f -perm -2000` |
| Find SGID directories | `sudo find /var /srv /home -type d -perm -2000` |

Do not grant SGID on executables belonging to sensitive groups just to solve access errors. Use least privilege. For team directories, decide on group membership, access permissions, umask/default ACL and deletion policy separately.

## 6. Interview questions and answers

**What are the two uses of SGID?** On executables it can change effective group identity; on directories it ensures group inheritance for new entries and SGID inheritance for new directories.

**Does SGID change existing files automatically?** No. A deliberate group ownership change is required for existing files.

**Does SGID grant group write permission?** No. Group ownership and access permissions are separate decisions.

**Why use `2770` for a private team directory?** The `2` enables group inheritance, `770` gives the owner and group full normal access, and others get none.

**What does uppercase `S` mean?** The SGID bit is set, but group execute is missing. It does not mean the special bit is cleared.

**What is the difference between SUID and SGID?** SUID changes effective user identity for executables; SGID changes effective group identity for executables and also provides group inheritance on directories.

## References

- [inode(7)](https://man7.org/linux/man-pages/man7/inode.7.html)
- [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)
- [chmod(1)](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [umask(2)](https://man7.org/linux/man-pages/man2/umask.2.html)
- [Microsoft Learn — WSL file permissions](https://learn.microsoft.com/en-us/windows/wsl/file-permissions)

---

**Remember:** SGID answers *whose group identity does this executable use, or which group should new files inherit?*
