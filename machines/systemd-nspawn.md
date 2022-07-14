systemd-nspawn
==============

## Basics

- see <distros/disk.md> to create a chroot or disk image with `mkosi`
  - or it can be manual
- `sudo systemd-nspawn -D <chroot>` to start shell as pid 1
  - this forks in a new namespace and execs `<chroot>/bin/bash`
  - `systemd-nspawn` waits for the shell to exit
- `--boot` searches init and starts it as pid 1 instead
- if starting a command that is not init or shell, one should specify `-a` as
  well
  - `-a` tells `systemd-nspawn` to start a stub init as pid 1, to reap the
    command
- user namespace
  - disabled by default unless `-U` is specified
  - for `-U` to work, `chown -R 65536.65536 <chroot>` first
    - it appears that `systemd-nspawn` uses the owner of `<chroot>` as the
      real user.  Without this step, the real user will still be root.
  - there are a few errors when combined with `--boot`
    - need network namespace!
    - I don't see the errors anymore though
- network namespace
  - disabled by default unless `-n` is specified
  - `-n` creates a veth pair; to configure manually,
    - in the host,
      - `sudo ip link set ve-chroot up`
      - `sudo ip addr add 192.168.0.1/24 dev ve-chroot`
      - set up snat
    - in the container,
      - `ip link set host0 up`
      - `ip addr add 192.168.0.2/24 dev host0`
      - `ip route add default via 192.168.0.1`
      - `echo "nameserver 8.8.8.8" >> /etc/resolv.conf`
- in short,
  - `sudo systemd-nspawn -D <chroot>` for chroot-like result
  - `sudo systemd-nspawn -bnUD <chroot>` for container-like result
    - `chown -R 65536.65536 <chroot>` first if the real user should be nobody
