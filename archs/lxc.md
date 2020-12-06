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
  - `lxc-start -n test`
  - this starts the container and runs `/sbin/init`
  - all processes in the container are regular processes and show up in the
    host `ps -ef` as well
  - but they are in a different pid ns
- get the shell
  - `lxc-attach -n test`
- stop the container
  - `lxc-stop -n test`
- debug
  - `lxc-start -n test -o log -l DEBUG`

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

- by default, the container uses host net namespace
- to have its own net namespace,
  - edit `/var/lib/lxc/<container>/config`
    - `lxc.net.0.type = veth`
    - `lxc.net.0.link = lxcbr0`
    - `lxc.net.0.flags = up`
  - veth is a virtual ethernet device that always comes in pairs
    - one end in host
    - one end in container ns
- on the host, create a bridge
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
- inside container
  - DHCP
  - or static
    - `ip addr add 192.168.5.2/24 dev eth0`
    - `ip route add default via 192.168.5.1 dev eth0`
- automatic host setup
  - create lxcbr0 bridge
    - create `/etc/default/lxc-net`
      - `USE_LXC_BRIDGE="true"`
    - `systemctl enable lxc-net`
    - it will create lxcbr0, run dnsmasq, and set up NAT
  - allow regular users to create veth pairs
    - create `/etc/lxc/lxc-usernet`
      - `<user> veth lxcbr0 4`

## Other Commands

- list the containers
  - `lxc-ls --fancy`

## Unprivileged Containers

- use new user namespace
  - container config should have
    - `lxc.idmap = u 0 100000 65536`
    - `lxc.idmap = g 0 100000 65536`
  - this invokes `newuidmap` and `newgidmap` to map [100000-165536] in parent
    namespace to [0-65536] in new namespace
  - for `newuidmap` and `newgidmap` to work, update /etc/sub{uid,gid} to
    something like
    - format: `<user>:<first-uid>:<count>`
    - example: `root:100000:65536` or `olv:100000:65536`
- network
  - `lxc.net.0.type = none` inherits host network and does not work
    - mounting sysfs results in permission deny because kernel requires
      `CLONE_NEWNET` as well
  - `lxc.net.0.type = empty` works
    - but no network
  - `lxc.net.0.type = veth` works
    - but require setup
- this is enough to start unprivileged containers as root
- to start as a regular user,
  - copy `/etc/lxc/default.conf` to `~/.config/lxc/default.confg`
  - update `/etc/pam.d/system-login` or `/etc/pam.d/common-session`
    - `session    optional   pam_cgfs.so -c freezer,memory,name=systemd,unified`
    - not needed with pure cgroup v2?
  - `chmod o+x $HOME`

## Mounts

- wayland socket
  - make sure container users have access to host `$XDG_RUNTIME_DIR/wayland-0`
    - including the socket and the all parent directories
  - add `lxc.mount.entry = /run/user/1000/wayland-0 mnt/wayland-0 none bind,create=file`
    - mount to /mnt becuase systemd over-mounts tmpfs to /run after the bind mount
  - in the container, try `XDG_RUNTIME_DIR=/mnt WAYLAND_DISPLAY=wayland-0 weston-flower`
