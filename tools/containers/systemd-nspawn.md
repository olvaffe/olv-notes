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
  - `-U` implies `--private-users=pick`
    - it maps `0..65535` in the container to `N...N+65535` in the host
    - `N` is determined by the owner of chroot
      - `chown -R 65536:65536 <chroot>` first
  - `-U` also implies `--private-users-ownership=auto`
    - once N is picked above, it chowns everything to be offset by N
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
    - `chown -R 65536:65536 <chroot>` first if the real user should be nobody
    - set up snat and net

## Minimal OS Tree

- minimal os tree without `-b`
  - there must be `/usr` in chroot
  - systemd-nspawn automatically creates and mounts special filesystems
    - `/root`
    - `/var`
    - `/var/log`
    - `/var/log/journal`
    - `/etc`
    - `/etc/localtime`
    - `/etc/resolv.conf`
    - `/proc`
    - `/sys`
    - `/dev`
    - `/tmp`
    - `/run`
- these directories/files are required additionally with `-b`
  - `/etc/os-release`
  - `/sbin/init`
    - or `/usr/lib/systemd/systemd` or `/lib/systemd/systemd`

## mkosi

- create chroot
  - `mkdir mkosi; cd mkosi`
  - `mkdir workspace cache`
  - `sudo mkosi -d debian -r testing -t directory -o chroot --cache cache \
     --workspace-dir $PWD/workspace`
  - unfortunately, must be in the same architecture
    - it should be better to just use the distro boostrapping tools directly
- create disk image
  - `-t gpt_btrfs` creates a disk image which has one gpt partition which is
    the rootfs in btrfs
  - `--root-size 10G` to set the rootfs size to 10G
- `systemd-nspawn`
  - `systemd-nspawn -D <chroot>` or `systemd-nspawn -i <image>` to get root
    shell
  - `systemd-nspawn -b ...` to run `init` as pid 1
    - to login, the image should have been created with `--password` or
      `--autologin`; or use the root shell to set a password
- create bootable disk image
  - `--bootable` adds another gpt partition for efi, and installs bootloader
    and kernel
  - bootable with `system-nspawn -bi <image>`, VMs, or real machines
- manually mount a disk image
  - `losetup -fP <image>` to attach
  - mount normally
  - `losetup -D` to detach
