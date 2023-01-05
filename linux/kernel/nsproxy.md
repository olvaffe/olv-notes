Kernel nsproxy
==============

## `nsproxy`

- `nsproxy` is a struct that holds all namespaces
  - `current->nsproxy`

## namespaces

- `clone` is similiar to `fork` and has more controls
  - e.g., setting `CLONE_THREAD` creates a thread
- `unshare` allows a process (or thread) to control its shared execution
  context without creating a new process (or thread)
- interesting namespace-related flags are
  - `CLONE_NEWCGROUP`
    - new cgroup namespace
    - require root
  - `CLONE_NEWIPC`
    - new ipc namespace for `sysvipc` and `mq_overview`
    - require root
  - `CLONE_NEWNET`
    - new net namespace
    - require root
    - initially only `lo`
    - each net device can only belong to a single net namespace
      - use `ip netns` to manage
  - `CLONE_NEWNS`
    - new mount namespace
    - require root
    - initial values are inherited from parent
  - `CLONE_NEWPID`
    - new pid namespace
    - require root
    - when used with `unshare`, only affects child processes created
      afterwards
      - the first child will have pid 1
    - `/proc` shows the pid namespace of the process who mounts it
      - normally, pid 1 should chroot and mount `/proc`
      - or, pid 1 can mount over `/proc` with new mount namespace
      - need user namespace to be able to mount
  - `CLONE_NEWTIME`
    - new time namespace for system clocks such as `CLOCK_MONOTONIC`
    - require root
    - can only be set with `unshare` and only affects child processes created
      afterwards
  - `CLONE_NEWUSER`
    - new user namespace for uids and gids
    - no priviledge required
    - until uid/gid mappings are set up, the process has unmapped uid/gid
      which gets overflowed to 65534
    - all capabilities are granted
      - note that `execve` will drop capabilities unless the uid is 0
  - `CLONE_NEWUTS`
    - new UTS namespace for host name and (unused) NIS domain name
    - root-only
    - initial values are inherited from parent
    - UTS stands for unix time sharing and comes from `utsname` of `uname`
