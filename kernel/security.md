Security
========

## seccomp

- kernel configs
  - `CONFIG_SECCOMP` is default on
  - `CONFIG_SECCOMP_FILTER` is also default on
- syscalls
  - `seccomp()` or the older/inferior `prctl(PR_SET_SECCOMP)`
  - only the calling thread is affected, unless `SECCOMP_FILTER_FLAG_TSYNC` is
    set
  - child processes inherit the settings (when forking is allowed)
  - new threads inherit the setting (when `clone` is allowed)
  - `execve` preserves the setting (when `execve` is allowed)
- operations
  - `SECCOMP_SET_MODE_STRICT`
    - the only system calls allowed are `read`, `write`, `_exit`, and
      `sigreturn`
    - no flags nor args
  - `SECCOMP_SET_MODE_FILTER`
    - must be preceded by `prctl(PR_SET_NO_NEW_PRIVS, 1)` unless root
    - `args` is a pointer to a `struct sock_fprog`, to pass a BPF program to
      the kernel for syscall filtering
    - flags can be
      - `SECCOMP_FILTER_FLAG_LOG` to log bpf return values specified by
        `/proc/sys/kernel/seccomp/actions_logged`
      - `SECCOMP_FILTER_FLAG_NEW_LISTENER` to return an fd to userspace which
        the bpf program can send notifications to
      - `SECCOMP_FILTER_FLAG_SPEC_ALLOW` to allow speculative store bypass
        mitigation
      - `SECCOMP_FILTER_FLAG_TSYNC` to change the filter for all threads
        rather than just the calling thread
    - bpf return values can be
      - `SECCOMP_RET_KILL_PROCESS` to terminate the process of the offending
        thread and send `SIGSYS`
      - `SECCOMP_RET_KILL_THREAD` to terminate the offending thread and send
        `SIGSYS`
      - `SECCOMP_RET_TRAP` to send `SIGSYS`
      - `SECCOMP_RET_ERRNO` to fail the syscall with the specified errno
      - `SECCOMP_RET_USER_NOTIF` to send a notification through an fd
      - `SECCOMP_RET_TRACE` to send  a notification to a `ptrace`-based tracer
      - `SECCOMP_RET_LOG` to log and allow the syscall
      - `SECCOMP_RET_ALLOW` to allow the syscall
      - any other value is treated as `SECCOMP_RET_KILL_PROCESS`
  - `SECCOMP_GET_ACTION_AVAIL`
    - returns available filter return actions
  - `SECCOMP_GET_NOTIF_SIZES`
    - returns the size of notifications
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
