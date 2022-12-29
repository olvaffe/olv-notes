Arch Linux
==========

## Installation

- Preparation
  - require an internet connection
    - `ip link`
    - static
      - `ip link set <iface> up`
      - `ip addr add 192.168.0.1/24 dev <iface>`
      - `ip route add default via 192.168.0.254`
    - dynamic: `dhcpcd <iface>`
    - wireless: `iwctl station <iface> connect <ssid>`
      - no easy way if the wireless adapter is not supported by the live image
  - system clock
    - `timedatectl set-ntp true`
  - partition disk with fdisk
    - see <disk.md>
    - 1M for BIOS boot partition (type 4), if bios
    - 260M for EFI system partition (type 1), esp
      - `mkfs.vfat -F32 <part>`
    - remainder for root partition
      - `mkfs.btrfs <part>`
      - `mkdir roots homes`
      - `btrfs subvolume create roots/current` for real root
      - `mkdir roots/current/{boot,home}`
      - `btrfs subvolume create homes/current` for real home
      - `btrfs subvolume set-default roots/current`
  - mount partitions to /mnt, /mnt/home, and /mnt/boot
- Bootstrap
  - update `/etc/pacman.d/mirrorlist`
  - `pacstrap /mnt base`
  - `genfstab -U /mnt >> /mnt/etc/fstab` and double-check
- Enter chroot
  - `arch-chroot /mnt`
  - `hwclock --systohc`
    - this updates `/etc/adjtime` so should be done in chroot
  - install more packages
    - `pacman -S linux linux-firmware`
    - `pacman -S btrfs-progs dosfstools`
    - `pacman -S dhcpcd iwd wpa_supplicant`, at most one of them should suffice
    - `pacman -S sudo vim`
    - `pacman -S grub`, if using BIOS
    - `pacman -S broadcom-wl-dkms`, or other out-of-tree drivers
  - uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen` and run `locale-gen`
  - `systemd-firstboot --prompt`, or
    - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
    - `echo LANG=en_US.UTF-8 > /etc/locale.conf`
    - `echo <host-name> > /etc/hostname`
    - `passwd` to set a password for root
  - create user
    - `useradd -m -G wheel,video <user>`
    - `passwd <user>`
    - `visudo`
  - traditionally, also
    - `echo -e '127.0.0.1\tlocalhost\n::1\t\tlocalhost' >> /etc/hosts`
      - replaced by `libnss_myhostname`
  - install bootloader
    - `bootctl install` for EFI
    - create `/boot/loader/entries/arch.conf`
      - `title	Arch Linux`
      - `linux	/vmlinuz-linux`
      - `initrd	/initramfs-linux.img`
      - `options root=...`
    - `echo 'default arch.conf' >> /boot/loader/loader.conf`
    - or, on a BIOS system,
      - `grub-install <dev>`
      - `grub-mkconfig -o /boot/grub/grub.cfg`
- Reboot

## Post-Installation

- login as user
  - `git clone https://github.com/olvaffe/olv-etc.git`
  - ./olv-etc/create-links
- network
  - for quick connection, see `Installation`
  - for wired, <../networks/systemd-networkd.md>
    - create `/etc/systemd/network/blah.network`
    - `systemctl enable --now systemd-networkd systemd-resolved`
  - for wireless,
    - if need built-in dhcp, create `/etc/iwd/main.conf`
      - `[General]`
      - `EnableNetworkConfiguration=true`
    - `systemctl enable --now iwd systemd-resolved`
  - for wireless adapter that requiers out-of-tree driver,
    - `dkms autoinstall`
    - `/etc/modules-load.d`
    - `/etc/modprobe.d/blacklist.conf`
- minimal packages
  - boot, `base linux linux-firmware`
    - also `btrfs-progs dosfstools`
    - also `zram-generator`
      - `echo '[zram0]' > /etc/systemd/zram-generator.conf`
      - `systemctl daemon-reload`
      - `zramctrl`
  - wifi, `iwd` or `wpa_supplicant`
    - also `networkmanager`
  - tools, `sudo vim man-db`
  - devel, `gcc git ctags meson pkgconf`
    - also `gdb debuginfod perf strace man-pages`
  - gui, `sway swayidle swaylock i3status mako polkit`
    - `alacritty google-chrome noto-fonts noto-fonts-cjk`
    - `light wl-clipboard wayland-utils`
    - `imv grim slurp`
    - `mesa mesa-utils vulkan-radeon vulkan-tools`
  - bluetooth, `bluez bluez-utils`
  - printer, `cups samsung-unified-driver-printer`
  - audio, `pipewire pipewire-pulse pavucontrol`
  - misc
    - `fakeroot` for makepkg
    - `bison flex python-mako wayland-protocols` for mesa
    - `cmake make` for cmake
    - `aarch64-linux-gnu-gcc` for cross-compile
      - also `qemu-user-static qemu-user-static-binfmt`
- x11
  - xwayland: `xorg-xwayland`
  - X11: `xorg xorg-xinit i3 xterm`
- 32-bit
  - uncomment the `[multilib]` section in `/etc/pacman.conf`
  - `pacman -S multilib-devel`
  - `pacman -S lib32-{mesa,libdrm,libunwind,libx11}`
  - `pacman -S steam`
    - `pacman -S ttf-liberation` if ui text is garbled
    - `pacman -S lib32-systemd` if no network

## Installation in crosvm

- create disk image
  - `crosvm create_qcow2 arch.qcow2 <size-in-bytes>`
- prepare iso, kernel and initramfs
  - download iso
  - extract kernel and initramfs under `arch/boot/x86_64` in iso
    - use loop mount or 7z
- start crosvm
  - `--rwdisk arch.qcow2 --disk <path-to-iso> -p archisodevice=/dev/vdb`
  - `--host_ip 192.168.0.1 --netmask 255.255.255.0 --mac 12:34:56:78:9a:bc`
  - `--disable-sandbox`
- installation
  - for network,
    - `ip addr add 192.168.0.2/24 dev enp0s5`
    - `ip route add default via 192.168.0.1`
  - `arch.qcow2` is `/dev/vda`
    - `/boot`, `linux`, `linux-firmware`, and bootloader not needed
    - but can still partition, format, and bootstrap normally for qemu

## Tidy Up an Existing Installation

- `pacman -Qeq` to get explicitly packages
- `pacman -D --asdeps` to mark them deps
- `pacman -D --asexplicit` to mark desired packages explicit
- `pacman -Qtdq` to get packages to remove

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
- manually
  - use `pacstrap` from `arch-install-scripts`
    - <https://github.com/archlinux/arch-install-scripts>

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
