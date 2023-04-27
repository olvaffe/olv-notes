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
