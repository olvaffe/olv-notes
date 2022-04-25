systemd-nspawn
==============

## Basics

- see <distros/disk.md> to create a chroot or disk image with `mkosi`
  - or it can be manual
- `sudo systemd-nspawn -D <chroot>` to start shell as pid 1
  - `--boot` searches init and starts it as pid 1 instead
  - if starting a command that is not init or shell, specify `-a` as well
    - this starts a stub init as pid 1, to reap processes
- user namespace
  - disabled by default unless `-U` is specified
  - for `-U` to work, `chown -R 65536.65536 <chroot>` first
  - `sudo systemd-nspawn -UD <chroot>` starts shell as a non-root user
    - which appears to be root in the container
  - there are a few errors when combined with `--boot`
    - network namespace!
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
