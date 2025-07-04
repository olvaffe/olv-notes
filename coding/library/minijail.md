minijail
========

## Main Functions

- `minijail_enter` causes the caller process to enter the jail
- `minijail_fork` forks and puts the subprocess in the jail
- `minijail_run` forks, puts the subprocess in the jail, and exec
  - this might require `LD_PRELOAD`
- There are many jailing parameters.  We only show the commonly used ones
  here.

## `minijail_enter`

- new mount namespace
  - `unshare(CLONE_NEWNS)`
  - requires `CAP_SYS_ADMIN`
- new net namespace
  - `unshare(CLONE_NEWNET)`
  - requires `CAP_SYS_ADMIN`
- mounts and bind mounts
- chroot
  - `pivot_root`, which replaces the root mount in the mount table
  - in contrast, `chroot` makes no change to the mount table
- remount proc ro
  - remount such that proc does not show the old pid namespace
- drop caps
  - `cap_set_proc`
- no new privs
  - `prctl(PR_SET_NO_NEW_PRIVS, ...)`
- seccomp
  - `seccomp(SECCOMP_SET_MODE_FILTER, ...)`
  - requires `CAP_SYS_ADMIN` or `PR_SET_NO_NEW_PRIVS`
    - the latter is to prevent an unpriviledged process from execing an suid
      executable with a malicious filter 

## `minijail_fork`

- new user and pid namespaces
  - parent calls `clone(CLONE_NEWPID | CLONE_NEWUSER | SIGCHLD)`
  - `SIGCHLD` means to send the signal when the child terminates
- set rlimits
  - parent calls `prlimit(child, ...)`
  - parent requires `CAP_SYS_RESOURCE` to change hard limits
- update uid/gid map
  - parent updates `/proc/<child>/[ug]id_map`
  - this remaps uid/gid of child in the new user namespace
  - without remapping, the uid/gid is 65534
- parent returns at this point
- child waits parent to do the above before continuing
- change uid/gid
  - `setresgid` and `setresuid`, if a remapping is set by the parent
  - because this is a new user namespace, all caps are granted
- `minijail_enter`
  - with remount proc ro disabled

## `minijail_run`

- parent does preparation works
  - sets up a pipe and marshals the jail to the pipe
  - sets up `LD_PRELOAD` to override `__libc_start_main` after child execs
- parent calls `minijail_fork`
- child calls `minijail_enter`
  - preceded by `minijail_preexec`
  - still enters new mount and net namespaces
  - but other jailings are disabled
- child execs
  - in a new user namespace, all caps are granted
  - after exec, all caps are dropped
  - man capabilities says that ambient caps are needed to keep caps
- child enters `fake_main` before `main`
- child demarshals the jail from the pipe
- child calls `minijail_enter` again
  - preceded by `minijail_preenter`
  - still mounts, chroot, drop caps, seccomp, etc.

## `minijail0` cmdline options

- `minijail0` forks and execs the specified program in a sandbox
- `-i` causes the parent process to exit
- `-u` changes the uid
- `-g` changes the gid
- `-G` inherits the supplementary groups of the uid specified by `-u`
- `-p` creates a pid namespace
- `-v` creates a mount namespace
- `-U` creates a user namespace
- `-N` creates a cgroup namespace
- `-l` creates an ipc namespace
- `-r` remounts `/proc` ro
- `-t` mounts tmpfs to `/tmp`
- `-f` writes pid of the child to the specified file
- `-P` chroots to the specified dir using `pivot_root`
- `-b` bind-mounts the specified paths
- `-k` mounts the specified paths
- `-R` sets rlimits
- `-n` sets `no_new_privs`
- `-S` enables seccomp
