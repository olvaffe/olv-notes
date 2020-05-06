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
- different net namespace
  - `network_namespaces(7)`
  - `nsenter -t <pid> -n ip link`
- different cgroup namespace
  - `cgroup_namespaces(7)`
  - `/sbin/init` thinks it is the root cgroup (in the new namespace)
  - from the host's view, lxc sets it up to be a new cgroup under the root
    cgroup

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
- container config
  - edit `/var/lib/lxc/<container>/config`
  - `lxc.net.0.type = veth`
  - `lxc.net.0.link = lxcbr0`
  - `lxc.net.0.flags = up`
- inside container
  - for wired host
    - dhcp
  - for wireless host
    - `ip addr add 192.168.5.2/24 dev eth0`
    - `ip route add default via 192.168.5.1 dev eth0`

## Other Commands

- list the containers
  - `lxc-ls --fancy`
- get the state of a container
  - `lxc-state test`
