ArchLinux
=========

## Installation

- Pre-installation
 - require an internet connection
   - `ip link`
   - wired: `dhcpcd <iface>`
   - wireless: `iwctl station wlan0 connect <SSID>`
     - no easy way if the wireless adapter is not supported by the live image
 - system clock
   - `timedatectl set-ntp true`
 - partition disk with fdisk
   - 260M for EFI system partition (type 1), esp
     - format esp to vfat
   - remainder for root partition
     - format to btrfs
     - `mkdir roots homes`
     - `btrfs subvolume create roots/current` for real root
     - `btrfs subvolume create homes/current` for real home
     - `btrfs subvolume set-default roots/current`
 - mount partitions to /mnt, /mnt/home, and /mnt/boot
- Installation
   - update `/etc/pacman.d/mirrorlist`
   - `pacstrap /mnt base linux linux-firmware vim`
     - and `iwd` or `dhcpcd` depending on wirelss or wired
     - or `wpa_supplicant` if `iwd` does not work
   - `genfstab -U /mnt >> /mnt/etc/fstab`
- Enter chroot
   - `arch-chroot /mnt`
   - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
   - `hwclock --systohc`
   - uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen` and run `locale-gen`
   - `echo LANG=en_US.UTF-8 > /etc/locale.conf`
   - `echo <host-name> > /etc/hostname`
   - `echo -e 127.0.0.1\tlocalhost >> /etc/hosts`
   - `echo -e ::1\tlocalhost >> /etc/hosts`
   - `echo -e 127.0.1.1\t<host-name> >> /etc/hosts`
   - `passwd` to set a password for root
   - `bootctl --path=/boot install` for EFI
     - set up `/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf`
     - optionally configure to update the bootloader automatically
     - on BIOS systems,
       - `pacman -S grub`
       - `grub-install /dev/$DISK`
       - `grub-mkconfig -o /boot/grub/grub.cfg`
- Reboot

## Installation in crosvm

- create disk image
  - `crosvm create_qcow2 arch.qcow2 <size-in-bytes>`
- prepare kernel and initramfs
  - download iso
  - extract kernel and initramfs
    - under `arch/boot/x86_64` in iso
    - use loop mount or 7z
- start crosvm
  - `--rwdisk arch.qcow2 --disk <path-to-iso> -p archisodevice=/dev/vdb`
  - `--host_ip 192.168.0.1 --netmask 255.255.255.0 --mac 12:34:56:78:9a:bc`
  - `--disable-sandbox`
- installation
  - `arch-qcow2` is `/dev/vda`
  - for network,
    - `ip addr add 192.168.0.2/24 dev enp0s5`
    - `ip route add default via 192.168.0.1`

## Network

- for temporary connection, see `Installation`
  - `/etc/iwd/main.conf`
    - `[General]`
    - `EnableNetworkConfiguration=true`
  - `systemctl restart iwd`
- for wireless adapter that requiers out-of-tree driver,
  - /etc/modules-load.d/
  - /etc/modprobe.d/blacklist.conf

## X11

- `pacman -S xorg xorg-xinit xterm`
- `pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji`
- `pacman -S i3`

## Development

- `pacman -S base-devel`
- `pacman -S git meson cmake`
- 32-bit
  - uncomment the [multilib] section in /etc/pacman.conf
  - `pacman -S multilib-devel`

## Wayland

## User

- add user
  - `useradd -m -G wheel <user>`
  - `passwd <user>`
  - `visudo`
- login as user
- `git clone https://github.com/olvaffe/olv-etc.git`
- ./olv-etc/create-links
- 

## Mesa

- `pacman -S mesa-demos vulkan-tools`
- `pacman -S python-mako wayland-protocols`
- 32-bit
  - `pacman -S lib32-{mesa,libdrm,libunwind,libx11}`

## Pacman

- `/var/lib/pacman/sync`
  - sync database
  - read-only, downloaded from the server
  - `$repo.db` is the package database for $repo
    - refreshed with `pacman -Sy`
  - `$repo.files` is the files database for $repo
    - refreshed with `pacman -Fy`
    - to list package files before installation
- `/var/lib/pacman/local`
  - local database
  - `$pkg/desc` is package description and metadata
  - `$pkg/files` is package files
  - `$pkg/mtree` is package file attributes
- `/var/cache/pacman/pkg`
  - cache of downloaded packages
- local database operations
  - `pacman -Q` queries the local database
  - `pacman -U` adds a downloaded package to the local database
    - `pacman -U $pkg.tar.xz`
  - `pacman -R` remove a package from the local database
  - `pacman -T` tests if a package is in the local database
  - `pacman -D` manipulates the local database 
- sync database operations
  - `pacman -S`
  - `pacman -F` queries the sync files database
- to estimate total installation size
  - `pacman -Qi | grep 'Installed Size' | grep MiB | awk '{s+=$4} END {print s}'`

## Bootstrap

- the bootstrap tarball, `archlinux-bootstrap-XXX.tar.gz`, is for chroot from
  a running system to pacstrap
  - the tarball is ~170M
  - untaring gives ~600M
- for size comparison,
  - `pacstrap` just the `base` package results in ~760M
    - ~630M after `pacman -Scc`
  - `pacstrap` just the `pacman` package results in ~560M
    - ~460M after `pacman -Scc`

## `base` package

- filesystem
  - directory strucutre (e.g., /root, /usr/bin)
  - basic links (e.g., /bin to /usr/bin)
  - some files in /etc (e.g., passwd, group, mtod)
- gcc-libs
  - gcc runtime libraries (`libgcc_s.so`, `libstdc++.so`, `libatomic.so`,
    `libasan.so`)
- glibc
  - headers
  - libraries (ld-linux-x86-64, libc, libdl, libm, libresolv, crt\*.o, etc)
  - tools (ldconfig, ldd, locale, locale-gen, iconv, etc)
- systemd
  - systemd, systemctl, timedatectl, networkctl, resolvect, loginctl,
    localectl, udevadm, etc.
- systemd-sysvcompat
  - init (symlink to systemd), reboot, shutdown, poweroff, halt (symlinks to
    systemctl)
- util-linux
  - agetty, blkid, dmesg, fdisk, kill, login, fsck, mkfs, mount, mountpoint,
    findmnt, rfkill, su, switch_root, nsenter, unshare, lsns, etc.
- bash
- shadow
  - grouadd, passwd, useradd, lastlog, newgidmap, newuidmap, etc.
- coreutils
  - ls, mv cp, sync, cat, chmod, chown, basename, cut, date, df, du, echo,
    env, chroot, etc.
- iputils
  - ping, tftpd, etc.
- iproute2
  - ip, ss, bridge
- procps-ng - ps, top, free, pidof, pkill
- psmisc
  - fuser, killall, pstree, etc.
- findutils
  - find, xargs
- grep
- sed
- gawk
- tar
- bzip2
- gzip
- xz
- file
- gettext
- pciutils
  - lspci
- licenses
- pacman
