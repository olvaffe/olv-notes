# Kernel nsproxy

## `nsproxy`

- `tsk->nsproxy` is the active namespaces of a task
  - the pointer may swap but the object is immutable
  - the object includes all except user ns
    - `uts_ns`, `ipc_ns`, `mnt_ns`, `net_ns`, `time_ns`, `cgroup_ns`
    - `pid_ns_for_children`, `time_ns_for_children`
    - user ns is in `tsk->cred->user_ns`
- `create_new_namespaces` creates a nsproxy object from an existing task
  - it shares or clones each namespace of the existing task
  - `CLONE_NEWNS` clones instead of shares `mnt_ns`
  - `CLONE_NEWUTS` clones instead of shares `uts_ns`
  - `CLONE_NEWIPC` clones instead of shares `ipc_ns`
  - `CLONE_NEWPID` clones instead of shares `pid_ns_for_children`
  - `CLONE_NEWCGROUP` clones instead of shares `cgroup_ns`
  - `CLONE_NEWNET` clones instead of shares `net_ns`
  - `CLONE_NEWTIME` clones instead of shares `time_ns_for_children`
  - `time_ns` is always shared
- `SYSCALL_DEFINE2(clone3) -> kernel_clone -> copy_process`
  - `copy_namespaces`
    - if all namespaces are shared, `get_nsproxy` shares the existing nsproxy
    - otherwise, `create_new_namespaces` creates a new nsproxy
- `SYSCALL_DEFINE1(unshare) -> ksys_unshare`
  - `unshare_nsproxy_namespaces`
    - `create_new_namespaces` creates a new nsproxy
- `SYSCALL_DEFINE2(setns)`
  - `prepare_nsset` creates a new nsproxy that shares all namespaces of
    `current->nsproxy`
  - `validate_nsset` updates the new nsproxy to share selected namespaces of
    the specified task
  - `commit_nsset` points `current->nsproxy` to the new nsproxy

## Namespaces

- `man user_namespaces`
  - The child process created by `clone(2)` with the `CLONE_NEWUSER` flag
    starts out with a complete set of capabilities in the new user namespace.
    Likewise, a process that creates a new user namespace using unshare(2) or
    joins an existing user namespace using setns(2) gains a full set of
    capabilities in that namespace.
  - On the other hand, that process has no capabilities in the parent (in the
    case of clone(2)) or previous (in the case of unshare(2) and setns(2))
    user namespace, even if the new namespace is created or joined by the root
    user (i.e., a process with user ID 0 in the root namespace).
  - Note that a call to execve(2) will cause a process’s capabilities to be
    recalculated in the  usual  way  (see  capabilities(7)).  Consequently,
    unless the process has a user ID of 0 within the namespace, or the
    executable file has a non‐empty inheritable capabilities mask, the process
    will lose all capabilities.
  - Having a capability inside a user namespace permits a process to perform
    operations (that require privilege) only on resources governed by that
    namespace.
  - Holding `CAP_SYS_ADMIN` within the user namespace that owns a process's
    mount namespace allows that process to create bind mounts and mount the
    following types of filesystems:
    - proc, sysfs, devpts, tmpfs, mqueue, bpf
    - to mount proc, needs pid namespace
    - to mount sysfs, needs user namespace
  - Holding `CAP_SYS_ADMIN` within the user namespace that owns a process's
    PID namespace allows (since Linux 3.8) that process to mount /proc
    filesystems.
  - When a nonuser namespace is created, it is owned by the user namespace in
    which the creating process was a member at the time of the creation of the
    namespace.
  - If `CLONE_NEWUSER` is specified along with other `CLONE_NEW*` flags in a
    single `clone(2)` or `unshare(2)` call, the user namespace is guaranteed
    to be created first, giving the child (clone(2)) or caller (unshare(2))
    privileges over the remaining namespaces created by the call.  Thus, it is
    possible for an unprivileged caller to specify this combination of flags.
  - When a user namespace is created, it starts out without a mapping of user
    IDs (group IDs) to the parent user namespace. The `/proc/[pid]/uid_map`
    and `/proc/[pid]/gid_map` files (available since Linux 3.5) expose the
    mappings for user and group IDs inside the user namespace for the process
    pid.  These files can be read to view the mappings in a user namespace and
    written to (once) to define the mappings.
  - In order for a process to write to the `/proc/[pid]/uid_map`
    (`/proc/[pid]/gid_map`) file, all of the following requirements must be
    met:
    - The writing process must have the `CAP_SETUID` (`CAP_SETGID`) capability
      in the user namespace of the process pid.
    - The writing process must either be in the user namespace of the process
      pid or be in the parent user namespace of the process pid.
    - The mapped user IDs (group IDs) must in turn have a mapping in the
      parent user namespace.
    - One of the following two cases applies:
      - Either the writing process has the CAP_SETUID (CAP_SETGID) capability
        in the parent user namespace.
    - No further restrictions apply: the process can make mappings to
      arbitrary user IDs (group IDs) in the parent user namespace.
    - i.e., use suid `newuidmap` (`newgidmap`)
      - Or otherwise all of the following restrictions apply:
    - The data written to `uid_map` (`gid_map`) must consist of a single
      line that maps the writing process's effective user ID (group ID) in
      the parent user namespace to a user ID (group ID) in the user
      namespace.
    - The writing process must have the same effective user ID as the
      process that created the user namespace.
    - In the case of `gid_map`, use of the setgroups(2) system call must
      first be denied by writing "deny" to the /proc/[pid]/setgroups file
      (see below) before writing to gid_map.
- `man pid_namespaces`
  - The first process created in a new namespace (i.e., the process created
    using `clone(2)` with the `CLONE_NEWPID` flag, or the first child created
    by a process after a call to `unshare(2)` using the `CLONE_NEWPID` flag)
    has the PID 1, and is the "init" process for the namespace (see init(1)).
    This process becomes the parent of any child processes that are orphaned
    because a process that resides in this PID namespace terminated (see below
    for further details).
  - A /proc filesystem shows (in the /proc/[pid] directories) only processes
    visible in the PID namespace of the process that performed the mount, even
    if the /proc filesystem is viewed from processes in other namespaces.
    - we see /proc mounted by the parent namespace and we should mount ours
- `man mount_namespaces`
  - The views provided by the /proc/[pid]/mounts, /proc/[pid]/mountinfo, and
    /proc/[pid]/mountstats files (all described in proc(5)) correspond to the
    mount namespace in which the process with the PID [pid] resides. (All of
    the processes that reside in the same mount namespace will see the same
    view in these files.)
  - A new mount namespace is created using either clone(2) or unshare(2) with
    the `CLONE_NEWNS` flag.  When a new mount namespace is created, its mount
    point list is initialized as follows:
    - If the namespace is created using clone(2), the mount point list of the
      child's namespace is a copy of the mount point list in the parent's
      namespace.
    - If the namespace is created using unshare(2), the mount point list of
      the new namespace is a copy of the mount point list in the caller's
      previous mount namespace.
  - Subsequent modifications to the mount point list (mount(2) and umount(2))
    in either mount namespace will not (by default) affect the mount point
    list seen in the other namespace (but see the following discussion of
    shared subtrees).
- `man uts_namespaces`
  - UTS namespaces provide isolation of two system identifiers: the hostname
    and the NIS domain name. These identifiers are set using sethostname(2)
    and setdomainname(2), and can be retrieved using uname(2), gethostname(2),
    and getdomainname(2). Changes made to these identifiers are visible to all
    other processes in the same UTS namespace, but are not visible to
    processes in other UTS namespaces.
  - When a process creates a new UTS namespace using clone(2) or unshare(2)
    with the `CLONE_NEWUTS` flag, the hostname and domain of the new UTS
    namespace are copied from the corresponding values in the caller's UTS
    namespace.
- `man cgroup_namespaces`
  - enable running systemd as pid 1 for an unprivileged user

## namespaces

- `clone` is similiar to `fork` and has more controls
  - e.g., setting `CLONE_THREAD` creates a thread
- `unshare` allows a process (or thread) to control its shared execution
  context without creating a new process (or thread)
- interesting namespace-related flags are
  - `CLONE_NEWUSER`
    - new user namespace for uids and gids
      - this is the only namespace flag that does not require root
      - this allows the process to become root in the new namespace, which
        allows it to set other namespace flags
    - capabilities
      - the process loses all capabilities in the parent user namespace
      - the process gains all capabilities in the new user namespace
      - for example,
        - `sudo mknod /dev/test c 1 3 && sudo rm /dev/test`
        - `sudo unshare -U -r`
        - `touch /dev/test && rm /dev/test`
          - this works because we are root in the parent namespace
        - `mknod /dev/test c 1 3 && sudo rm /dev/test`
          - this fails because we have lost all caps
      - note that `execve` will drop all capabilities in the new user
        namespace again unless the uid is 0
        - this is unrelated to namespace but the behavior of `execve`
        - `man capabilities`
    - uid/gid mappings
      - `uid_map`/`gid_map`
        - can be written once
        - map uids/gids in the parent namespace into the new namespace
        - there are 1:1 relations
        - permissions
          - must have `CAP_SETUID`/`CAP_SETGID` in the new namespace
          - unless mapping the effective uid/gid in the parent namespace, must
            have `CAP_SETUID`/`CAP_SETGID` in the parent namespace
            - how?  `CLONE_NEWUSER` drops all capabilities in the parent
              namespace
            - `newuidmap` has suid bit
      - if a process has uid/gid in the parent namespace
        - if uid/gid is mapped, it has the mapped uid/gid in the new namespace
        - if uid/gid is unmapped, it has the overflowed 65534/65534 in the new
          namespace
        - `setuid`/`setgid` fails with EINVAL if the new uid/gid in the new
          namespace has no mapping
  - `CLONE_NEWCGROUP`
    - new cgroup namespace
  - `CLONE_NEWIPC`
    - new ipc namespace for `sysvipc` and `mq_overview`
  - `CLONE_NEWNET`
    - new net namespace
    - initially only `lo`
    - each net device can only belong to a single net namespace
      - use `ip netns` to manage
  - `CLONE_NEWNS`
    - new mount namespace
    - initial values are inherited from parent
  - `CLONE_NEWPID`
    - new pid namespace
    - when used with `unshare`, only affects child processes created
      afterwards
      - the first child will have pid 1
    - `/proc` shows the pid namespace of the process who mounts it
      - normally, pid 1 should chroot and mount `/proc`
      - or, pid 1 can mount over `/proc` with new mount namespace
      - both chroot and mount require root
        - user namespace to the rescue if unprivileged
  - `CLONE_NEWTIME`
    - new time namespace for system clocks such as `CLOCK_MONOTONIC`
    - can only be set with `unshare` and only affects child processes created
      afterwards
  - `CLONE_NEWUTS`
    - new UTS namespace for host name and (unused) NIS domain name
    - initial values are inherited from parent
    - UTS stands for unix time sharing and comes from `utsname` of `uname`

## `nsenter`

- `nsenter` makes use of `setns` syscall
- different user namespace
  - `user_namespaces(7)`
  - `nsenter -t <pid> -U id`
- different pid namespace
  - `nsenter -t <pid> -p kill some-pid`
  - ps scans /proc and its output depends on mnt namespace
  - reboot(2) in a pid namespace kills pid 1
- different mnt namespace
  - `nsenter -t <pid> -m /bin/ls`
- different uts namespace
  - `nsenter -t <pid> -u hostname`
- different net namespace
  - `network_namespaces(7)`
  - `nsenter -t <pid> -n ip link`
- different cgroup namespace
  - `cgroup_namespaces(7)`
  - `/sbin/init` thinks it is the root cgroup (in the new namespace)
  - from the parent's view, it is a new cgroup under the root cgroup
- `lsns` or `/proc/<pid>/ns`
