ArchLinux
=========

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
   - 256M for EFI system partition (type 1), esp
     - format esp to vfat
   - remainder for root/home partition
     - format to btrfs
     - `mkdir roots homes`
     - `btrfs subvolume create roots/current` for real root
     - `btrfs subvolume create homes/current` for real home
     - `btrfs subvolume set-default roots/current`
 - mount partitions to /mnt, /mnt/home, and /mnt/boot
- Installation
   - update `/etc/pacman.d/mirrorlist`
   - `pacstrap /mnt base linux linux-firmware vim iwd systemd-resolvconf`
   - `genfstab -U /mnt >> /mnt/etc/fstab`
- Enter chroot
   - `arch-chroot /mnt`
   - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
   - `hwclock --systohc`
   - uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen` and run
     `locale-gen`
   - `echo <host-name> > /etc/hostname`
   - `echo -e 127.0.0.1\tlocalhost >> /etc/hosts`
   - `echo -e ::1\tlocalhost >> /etc/hosts`
   - `echo -e 127.0.1.1\t<host-name> >> /etc/hosts`
   - `passwd` to set a password for root
   - `bootctl --path=/boot install` for EFI
     - set up `/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf`
     - optionally configure to update the bootloader automatically
- Reboot

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
- `pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoj`
- `pacman -S i3`

## User

- add user
  - `useradd -m -G wheel <user>`
  - `passwd <user>`
  - `visudo`
- login as user
- `git clone https://github.com/olvaffe/olv-etc.git`
- ./olv-etc/create-links
- 

## Development

- `pacman -S base-devel`
- `pacman -S git meson cmake`

## Wayland
