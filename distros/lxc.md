Linux Containers
================

## Basics

- check system support
  - `lxc-checkconfig`
- create a container named `test` from the `download` template
  - `lxc-create -n test -t download`
  - the container is under `/var/lib/lxc/test`
  - it executes `/usr/share/lxc/templates/lxc-download`
  - images are downloaded from `https://images.linuxcontainers.org/`
- start the container
  - `lxc-start test`
  - this starts the container and runs `/sbin/init`
  - all processes in the container are regular processes and show up in the
    host `ps -ef` as well
  - but they are in a different pid ns
- get the shell
  - `lxc-attach`
- stop the container
  - `lxc-stop stop`

## Shells

- lxc-attach
  - `lxc_attach` calls `lxc_cmd_get_init_pid` to get the container's init pid,
    finds out the ns, openpty, and forks
  - the parent becomes the pty master
  - the child calls `lxc_attach_to_ns` to enter the ns, points
    stdin/stdout/stderr to the pty slave, and execs the shell
- lxc-console
  - lxc-start spawns and the child calls openpty multiple times.  lxc-start
    receives the master fds using `lxc_recv_ttys_from_child`
  - lxc-console requests one of the master fds

## Networks

- on the host
  - `ip link add name lxcbr0 type bridge`
  - `ip link set lxcbr0 up`
  - for wired host
    - `ip link set eth0 master lxcbr0`
  - for wireless host
    - `ip addr add 192.168.5.1/24 dev lxcbr0`
    - `sysctl net.ipv4.ip_forward=1`
    - `iptables -A POSTROUTING -t nat -j MASQUERADE`
    - optionally, start a DNS/DHCP server
      - `dnsmasq --conf-file= -a 192.168.5.1 -F 192.168.5.2,192.168.5.254`
- container config
  - edit `/var/lib/lxc/<container>/config`
  - `lxc.net.0.type = veth`
  - `lxc.net.0.link = lxcbr0`
  - `lxc.net.0.flags = up`
  - veth is a virtual ethernet device that always comes in pairs
    - one end in host
    - one end in container ns
- inside container
  - DHCP
  - or static
    - `ip addr add 192.168.5.2/24 dev eth0`
    - `ip route add default via 192.168.5.1 dev eth0`

## Other Commands

- list the containers
  - `lxc-ls --fancy`
- get the state of a container
  - `lxc-state test`

## `unshare`

- `strace unshare`
  - `unshare(0)`
  - `execve("/bin/bash", ...)`
- use strace to see what the various options do
- `man 2 clone` first
  - fork a child process
  - most of the "execution context" are shared with the child process
  - `CLONE_NEW*` speicifes which parts of the execution context are not shared
    - `CLONE_NEWUSER` puts the child in a new user namespace
- man 2 unshare
  - unshare parts of the execution context of the calling process (with it
    sparent)
  - in other words, the calling process enters new namespaces
- man 1 share
  - `CLONE_NEWUSER`, or `-U`
    - the caller obtains a full set of capabilities in the new namespace
    - almost always need `-U`
      - `unshare -p`: Operation not permitted
      - `unshare -Up`: ok
    - `-r` is even better
      - it modifies `/proc/self/uid_map` such that the uid becomes 0
      - `man subuid`
  - `CLONE_NEWPID`, or `-p`
    - The calling process is not moved into the new namespace.  The first
      child created by the calling process will have the process ID 1 and will
      assume the role of init(1) in the new namespace
    - always use with `-f` to fork pid 1
    - always use with `--mount-proc` to fix up /proc
  - `CLONE_NEWNS`, or `-m`
    - If the (mount) namespace is created using unshare(2), the mount point
      list of the new namespace is a copy of the mount point list in the
      caller's previous mount namespace.
    - `-m` also mounts root with `MS_REC|MS_PRIVATE` to make all inherited
      mount points private
    - now we can bind mount like crazy
    - implied by `--mount-proc`

## Namespaces

- `lsns` or `/proc/<pid>/ns`
- container's `/sbin/init` is run in differernt namespaces
- different mnt namespace
  - `mount_namespaces(7)`
  - `nsenter -t <pid> -m /bin/ls`
- different uts namespace
  - `uts_namespaces(7)`
  - `nsenter -t <pid> -u hostname`
- different ipc namespace
  - `ipc_namespaces(7)`
- different pid namespace
  - `pid_namespaces(7)`
  - `nsenter -t <pid> -p kill some-pid`
  - ps scans /proc and its output depends on mnt namespace
  - reboot(2) in a pid namespace kills pid 1
- different net namespace
  - `network_namespaces(7)`
  - `nsenter -t <pid> -n ip link`
- different cgroup namespace
  - `cgroup_namespaces(7)`
  - `/sbin/init` thinks it is the root cgroup (in the new namespace)
  - from the host's view, lxc sets it up to be a new cgroup under the root
    cgroup
- different user namespace, when running unprivileged containers
  - `user_namespaces(7)`
  - root in container is nobody-like outside

## Unprivileged Internals

- `newuidmap`
  - parses `/etc/subuid`
    - `<username>:<first_subid>:<subid_count>`

## Unprivileged Containers

- update /etc/sub{uid,gid} to something like
  - `<user>:100000:65536`
- update /etc/lxc/default.conf
  - `lxc.idmap = u 0 100000 65536`
  - `lxc.idmap = g 0 100000 65536`
- update /etc/pam.d/system-login
  - `session    optional   pam_cgfs.so -c freezer,memory,name=systemd,unified`
- we can start, as root, unprivileged containers with the changes above
- to start containers as a regular user,
  - update `/etc/lxc/lxc-usernet` with `<user> veth lxcbr0 10`
  - copy `/etc/lxc/default.conf` to `~/.config/lxc/default.confg`
