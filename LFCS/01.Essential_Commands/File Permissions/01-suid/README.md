# SUID — Set User ID

[← Main guide](../README.md) · [Next: SGID →](../02-sgid/README.md)

**Special octal digit:** `4` | **Symbol:** `s` / `S` in owner execute position | **Typical mode:** `4755`

## 1. The kid-friendly analogy: the principal's key

A student needs to update their address in a school records room. Only the principal has the key. Instead of giving every student that key, the principal provides a trusted machine that checks the student's identity and updates the allowed record on their behalf.

The student remains a student. The machine can use the principal's authority, but it must enforce its own rules.

| School story | Linux equivalent |
| --- | --- |
| Student | User executing a program |
| Principal | Executable's owner |
| Trusted machine | Executable with SUID enabled |
| Key | Effective user identity |
| Records room | Protected system data |

**Core idea:** SUID can give a running executable the effective user ID (EUID) of its file owner, instead of the EUID it would otherwise inherit from its caller.

## 2. Real-world Linux scenario: `passwd`

A normal user cannot directly update password hashes in `/etc/shadow`. Yet that user must be able to change their own password. A common solution is a trusted root-owned SUID program: `/usr/bin/passwd`.

Inspect the existing files; do not change them:

```bash
ls -l /usr/bin/passwd /etc/shadow
stat -c '%a %A %U:%G %n' /usr/bin/passwd
```

Typical output (permissions can vary by distribution):

```text
-rwsr-xr-x 1 root root   ... /usr/bin/passwd
-rw-r----- 1 root shadow ... /etc/shadow
4755 -rwsr-xr-x root:root /usr/bin/passwd
```

```text
Normal user (real UID 1000)
          |
          | launches passwd
          v
Root-owned executable with SUID (4755)
          |
          | kernel sets effective UID to file owner's UID (0)
          v
passwd validates the request and uses authorized privileges
          |
          v
Program exits; caller's original shell identity is unchanged
```

| Identity | Meaning | Example |
| --- | --- | --- |
| Real UID (RUID) | Identity associated with the caller | `1000` |
| Effective UID (EUID) | Effective identity used for many privilege decisions | `0` for root |
| Saved set-user-ID | A saved credential a program can use under defined rules to switch effective identity | Initially copied from the post-exec EUID |

Linux normally uses filesystem credentials (usually aligned with effective IDs), group membership, capabilities and other checks for file access. SUID does not bypass all security rules. The kernel changes the executable's credentials; the program is responsible for limiting what it allows. A poorly designed privileged executable can be dangerous.

Other traditional examples include `su`, `sudo`, and `chsh`; actual installed modes and privilege mechanisms depend on the distribution. `sudo` and SUID are not identical: `sudo` is a policy-driven command, whereas SUID is a filesystem mode bit.

## 3. Hands-on WSL lab: observe effective UID

**Goal:** See that ownership alone changes nothing, then see what changes when SUID is enabled. Use a copy of the native `id` executable owned by `nobody`, not root. No new accounts or groups are created.

Run all steps in the same Bash session. Start as a regular user with sudo access when possible. If already root, the helper runs commands directly; the real UID in your results will then be `0`.

<!-- lab-start -->
### Step 1 — Create an isolated playground

```bash
as_root() { if [ "$(id -u)" -eq 0 ]; then "$@"; else sudo "$@"; fi; }
mkdir -p "$HOME/linux-permissions-lab"
LAB_DIR=$(mktemp -d "$HOME/linux-permissions-lab/suid.XXXXXX")
cd "$LAB_DIR"
pwd
id
getent passwd nobody
findmnt -T . -o TARGET,FSTYPE
```

Stop if any setup command fails or the directory is on a Windows mount. The test uses the existing unprivileged `nobody` account, usually UID 65534.

### Step 2 — Establish a baseline

```bash
cp /usr/bin/id "$LAB_DIR/demo-id"
chmod 0755 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

Example (regular user Rahul):

```text
-rwxr-xr-x 1 rahul rahul ... demo-id
uid=1000(rahul) gid=1000(rahul) groups=1000(rahul)
```

### Step 3 — Change ownership, but not SUID

```bash
as_root chown nobody "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

Example:

```text
-rwxr-xr-x 1 nobody rahul ... demo-id
uid=1000(rahul) gid=1000(rahul) groups=1000(rahul)
```

The executable's owner changed, but its running identity did not. Ownership alone does not activate SUID.

### Step 4 — Enable SUID and observe the result

```bash
as_root chmod u+s "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
stat -c '%a %A %U:%G %n' "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
"$LAB_DIR/demo-id" -ru
"$LAB_DIR/demo-id" -u
whoami
```

Example:

```text
-rwsr-xr-x 1 nobody rahul ... demo-id
4755 -rwsr-xr-x nobody:rahul .../demo-id
uid=1000(rahul) gid=1000(rahul) euid=65534(nobody) groups=1000(rahul)
1000
65534
rahul
```

`id -ru` shows the real UID; `id -u` shows the effective UID. The shell's `whoami` remains unchanged. The equivalent numeric command is `chmod 4755 demo-id` when executed by the file owner or an administrator.

### Step 5 — Understand `s` vs. `S`

```bash
as_root chmod 4755 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
as_root chmod 4644 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

Expected modes and intentional error:

```text
-rwsr-xr-x
-rwSr--r--
bash: .../demo-id: Permission denied
```

| Symbol at owner execute position | SUID bit | Owner execute bit |
| --- | --- | --- |
| `x` | Off | On |
| `-` | Off | Off |
| `s` | On | On |
| `S` | On | Off |

`4644` has no execute bits at all, so it cannot be launched directly. An uppercase `S` alone does not prove the file is unexecutable: group or others may still have execute permission in a different mode, and SUID may still take effect for an authorized caller.

### Step 6 — Remove SUID and verify

```bash
as_root chmod u-s "$LAB_DIR/demo-id"
as_root chmod 0755 "$LAB_DIR/demo-id"
ls -l "$LAB_DIR/demo-id"
"$LAB_DIR/demo-id"
```

The mode returns to `0755`, and the executable no longer changes the effective UID.

### Step 7 — Clean up

```bash
as_root chmod u-s "$LAB_DIR/demo-id"
rm -- "$LAB_DIR/demo-id"
cd "$HOME/linux-permissions-lab"
rmdir -- "$LAB_DIR"
```
<!-- lab-end -->

## 4. WSL gotchas and troubleshooting

Use native Linux storage (`/home`, `/root`, or a suitable `/tmp`), not `/mnt/c` or `/mnt/d`. Windows DrvFs metadata is normally disabled by default, and Windows permissions still apply when enabled. WSL 2 is preferable for experiments with Linux kernel behavior.

Check while inside the lab:

```bash
findmnt -T . -o TARGET,FSTYPE,OPTIONS
grep '^NoNewPrivs:' /proc/$$/status
```

If `ls -l` displays `s` but the EUID does not change, check for these conditions:

| Cause | Explanation |
| --- | --- |
| `nosuid` mount | Blocks set-ID credential elevation at execution |
| `no_new_privs` | Prevents `execve` from gaining new privileges through set-ID bits |
| Tracing/debugging | Set-ID changes may be suppressed when the process is ptraced |
| Interpreted script | Linux ignores SUID and SGID bits on interpreter scripts |
| Program drops privileges | The executable can deliberately change its effective identity after starting |
| Caller already has owner's UID | No visible change in effective UID |
| Bit unexpectedly disappeared | Ownership changes and some writes may clear set-ID bits; set ownership first, then set the bit |

Check for `noexec` too if execution itself fails. Do not remount or weaken host security settings just to force a lab to work.

## 5. Admin command reference and security

| Task | Command |
| --- | --- |
| Enable SUID, preserving other permission bits | `chmod u+s file` |
| Set a complete example mode | `chmod 4755 file` |
| Remove SUID | `chmod u-s file` |
| Inspect numeric/symbolic mode | `stat -c '%a %A %U:%G %n' file` |
| List SUID regular files | `sudo find /usr -type f -perm -4000` |
| Check real/effective UID | `id -ru` / `id -u` |

Never set SUID on a shell, editor, interpreter or general-purpose command simply to make a permission problem disappear. Root-owned SUID executables deserve careful auditing, trusted code, restrictive write access, and the minimum necessary privileges. SUID has no ordinary directory behavior on Linux.

## 6. Interview questions and answers

**What is SUID and why is it necessary?** It allows a supported executable to run with its owner's effective UID. Trusted programs can perform limited privileged operations without making their callers administrators.

**Why does `passwd` commonly have SUID?** It needs controlled access to protected password data while allowing a regular user to change their own password.

**Does `chown root` alone make a program run as root?** No. File ownership alone does not change an executing process's EUID.

**What is the difference between `s` and `S`?** Both show that SUID is set. Only `s` also shows that owner execute is set.

**Does SUID work on Bash scripts?** Linux ignores set-ID bits on interpreted scripts. Use a supported native executable for this experiment.

**Does SUID permanently change the caller's user?** No. The executed program's credentials change, not those of its parent shell.

## References

- [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)
- [credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html)
- [inode(7)](https://man7.org/linux/man-pages/man7/inode.7.html)
- [Microsoft Learn — WSL file permissions](https://learn.microsoft.com/en-us/windows/wsl/file-permissions)

---

**Remember:** SUID answers *whose effective user identity does this executable use?* It does not grant permanent privileges to the caller.
