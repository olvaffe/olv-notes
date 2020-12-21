Security
========

## seccomp

- `CONFIG_SECCOMP` is default on
- `seccomp()` or `prctl(PR_SET_SECCOMP)`
- `SECCOMP_SET_MODE_STRICT`
  - the only system calls allowed are `read`, `write`, `_exit`, and
    `sigreturn`
- `SECCOMP_SET_MODE_FILTER`
  - system calls allowed are define by a caller-defined BPF program
  - must be preceded by `prctl(PR_SET_NO_NEW_PRIVS, 1)`
- a BPF program looks like
  - `BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, arch))`
    - load arch to accumulator
  - `BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0)`
    - if x86-64, jump 1 otherwise jump 0 instruction
  - `BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL)`
    - kill the calling thread
  - `BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr))`
    - load syscall nr to accumulator
  - `BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1)`
    - if `__NR_read`, jump 0 otherwise jump 1 instruction
  - `BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW)`
    - allow syscall
  - `BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL)`

## SELinux

- Mandatory Access Controls (MACs)
  - separate from other access controls such as filesystem permissions or sudo
  - can restrict a root
- `CONFIG_SECURITY_SELINUX` is default off
  - distros usually have it on
  - but userspace tools might not have `libselinux` support
  - and selinux policies might be missing
- to enable selinux in kernel, pass `security=selinux selinux=1`
- `pam_selinux` sets up security context for the session
- reference policy
  - `make bare`
  - `make conf`
  - `make policy`
  - `make install`
  - `make load`
- `/etc/selinux/config`
  - `SELINUX=permissive`
  - `SELINUXTYPE=refpolicy`
- userspace packages
  - `checkpolicy`
    - `checkpolicy`
    - `checkmodule`
  - `policycoreutils`
    - `fixfiles`
    - `load_policy`
    - `restorecon`
    - `restorecon_xattr`
    - `setfiles`
    - `secon`
    - `genhomedircon`
    - `semodule`
    - `sestatus`
    - `setsebool`

## Others

- SMACK
- TOMOYO
- AppArmor
- Yama
- lockdown
- bpf
