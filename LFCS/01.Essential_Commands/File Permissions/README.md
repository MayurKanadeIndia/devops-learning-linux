# Linux Special Permissions: SUID, SGID & Sticky Bit

A practical, beginner-friendly Linux permission guide for Ubuntu and CentOS on Windows Subsystem for Linux (WSL). Learn the **why**, understand the kernel behavior, and prove each concept with reproducible terminal experiments.

> **Learning goal:** Explain special permissions in an interview, configure them correctly, and troubleshoot them as a Linux/DevOps engineer — without memorizing commands blindly.

## Learning path

| Lesson | What you will learn | Hands-on experiment |
| --- | --- | --- |
| [01 — SUID](01-suid/README.md) | Real vs. effective user identity; controlled privileged operations | Execute a copy of `id` owned by an unprivileged account |
| [02 — SGID](02-sgid/README.md) | Effective group identity; group inheritance in shared directories | Compare normal and SGID directories; execute a group-owned binary |
| [03 — Sticky Bit](03-sticky-bit/README.md) | Ownership-based deletion and rename restrictions | Compare a shared directory with `0777` and `1777` using two test users |

## 1. Foundation: ordinary Linux permissions

A Linux file has an owner, a group, and permission bits for owner, group and others. Inspect them with `ls -l`, or use `ls -ld` for the directory itself rather than its contents.

```text
-rwxr-xr--
│└─┬┘└─┬┘└─┬┘
│ owner group others
└─ file type (- = regular file, d = directory)
```

| Bit | Octal value | On a regular file | On a directory |
| --- | ---: | --- | --- |
| `r` | 4 | Read contents | List entry names |
| `w` | 2 | Modify contents | Create, remove or rename entries (normally also needs `x`) |
| `x` | 1 | Execute | Traverse/search and access known entries |

For example, `0755` means owner `rwx` (7), group `r-x` (5), others `r-x` (5), with no special bits. The special permission digit is placed before the three ordinary octal digits.

## 2. Master comparison

| Feature | SUID | SGID | Sticky Bit |
| --- | --- | --- | --- |
| Leading octal digit | `4` | `2` | `1` |
| Symbol position | Owner execute | Group execute | Others execute |
| Symbols | `s` / `S` | `s` / `S` | `t` / `T` |
| Enable / disable | `chmod u+s` / `u-s` | `chmod g+s` / `g-s` | `chmod +t` / `-t` |
| Executable effect | Effective UID becomes file owner's UID | Effective GID becomes file group's GID (when group execute is set) | No standard modern Linux effect |
| Directory effect on Linux | No usual effect | New entries inherit group; new directories inherit SGID | Restrict deletion and renaming of entries |
| Typical mode | `4755` | `2755` binary / `2770` directory | `1777` directory |
| Real example | `/usr/bin/passwd` | `/usr/bin/wall` on some systems; shared project directories | `/tmp` |

```text
SUID         -rwsr-xr-x   4755
                ^ owner execute position
SGID         drwxrws---   2770
                   ^ group execute position
Sticky Bit   drwxrwxrwt   1777
                      ^ others execute position
```

Lowercase `s` or `t` means the special bit AND its corresponding execute bit are set. Uppercase `S` or `T` means the special bit is set, but that particular execute bit is not. Uppercase does not mean that the special bit itself is off.

### Combine special bits

| Leading digit | Special permissions |
| ---: | --- |
| 0 | None |
| 1 | Sticky Bit |
| 2 | SGID |
| 3 | SGID + Sticky Bit |
| 4 | SUID |
| 5 | SUID + Sticky Bit |
| 6 | SUID + SGID |
| 7 | All three |

Not every combination makes sense on every file type.

### A realistic DevOps design: `3770`

Suppose a team needs a private shared artifact directory. The administrator wants consistent group ownership and wants to prevent ordinary teammates from deleting each other's artifacts.

```text
Directory owner: root
Directory group: devops
Mode:            3770 (drwxrws--T)
```

SGID (`2`) assigns the team group to new entries. Sticky Bit (`1`) restricts removal and renaming. Ordinary permissions (`770`) limit access to owner and group. A suitable umask or default ACL is still required if the team needs group-writable files. This is not a good fit if the build process must replace teammates' files by renaming or deleting them.

## 3. Lab environment and safety

- Ubuntu or CentOS on WSL 2 is recommended. Use Bash and the native Linux filesystem, not `/mnt/c` or `/mnt/d`.
- Run each lab in one terminal session, preferably as a non-root user with `sudo`. Each lab defines an `as_root` helper for CentOS installations that start as root and do not have `sudo`.
- Run code blocks in sequence. If setup fails, stop and resolve the error before proceeding. Outputs are examples; usernames, GIDs, modes affected by local ACLs, and error text can differ.
- Labs create isolated directories with `mktemp`, avoid root-owned SUID demonstration binaries, and provide explicit cleanup. The Sticky Bit lab creates two temporary, password-locked accounts; the SGID lab creates one temporary group. These require administrator authorization.
- Do not change actual `/tmp`, `/etc/shadow`, `/usr/bin/passwd`, or other production files to experiment. Use the disposable copies and directories only.

Check the WSL version from **Windows PowerShell**:

```powershell
wsl -l -v
```

Check the Linux filesystem containing your current directory:

```bash
findmnt -T . -o TARGET,FSTYPE,OPTIONS
```

Windows-mounted drives use DrvFs permission translation. Linux metadata is normally disabled by default there, and Windows access controls still matter when metadata is enabled. Do not assume that a mode displayed by `ls` proves that executable credential changes or directory protections work on that filesystem.

## 4. Administrator's command reference

The following are inspection commands; they do not change permissions:

```bash
ls -l /usr/bin/passwd
ls -ld /tmp /var/tmp /dev/shm
stat -c '%a %A %U:%G %n' /usr/bin/passwd /tmp
id                            # real/effective IDs and groups
sudo find /usr -type f -perm -4000   # SUID files
sudo find /usr -type f -perm -2000   # SGID files
sudo find /var /srv /home -type d -perm -2000  # SGID directories
sudo find /tmp /var/tmp -type d -perm -1000    # Sticky directories
```

Use `-perm -4000` (and similarly for the other bits) to match entries that have the specified bit set even when other bits are also present. A search may report missing paths or denied access; examine errors during an audit rather than automatically hiding them. On a root shell, omit `sudo`.

## 5. Interview revision

**Why can a normal user change their password?** A trusted SUID program such as `passwd` can perform authorized operations using its owner's effective UID, typically root, while enforcing application-specific checks.

**Does SGID automatically make files group-writable?** No. It controls inherited group ownership. Initial permission bits depend on the requested creation mode, umask and any default ACL.

**Why can a read-only file be deleted?** Deletion changes the directory entry. Parent directory permissions, ownership restrictions and other security checks determine whether deletion is allowed; the file's ordinary write bit is not required.

**Why is `/tmp` normally `1777` instead of `777`?** Both allow users to create entries. `1777` adds an ownership restriction that prevents ordinary users from deleting or renaming entries owned by other users.

**What do uppercase `S` and `T` mean?** The respective special permission is set, but the execute bit at that position is absent. Other classes may still have execute permission. For executable SGID, group execute must be set for the effective group change.

**Can you combine SGID and Sticky Bit?** Yes, for example `3770` on a restricted shared directory. Group inheritance and deletion restrictions address different problems.

## 6. Next topic: umask and ACLs

These three bits do not replace ordinary permissions. Umask determines which permission bits are masked off at creation (subject to default ACL behavior), and ACLs allow more granular access rules for particular users and groups. Learn those next to build a complete permissions model.

## References

- [Linux inode(7): file types and mode bits](https://man7.org/linux/man-pages/man7/inode.7.html)
- [Linux execve(2): executable identity changes and exceptions](https://man7.org/linux/man-pages/man2/execve.2.html)
- [GNU Coreutils: mode structure](https://www.gnu.org/software/coreutils/manual/html_node/Mode-Structure.html)
- [GNU chmod manual](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [Microsoft Learn: file permissions for WSL](https://learn.microsoft.com/en-us/windows/wsl/file-permissions)

---

**Memory hook:** SUID = whose user identity? SGID = whose group identity or inherited group? Sticky Bit = who may delete or rename?
