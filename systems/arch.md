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
   - `dhcpcd <iface>`
   - no easy way if the wireless adapter is not supported by the live image
 - system clock
   - `timedatectl set-ntp true`
 - partition disk with fdisk
   - 256M for EFI system partition (type 1), esp
   - 32G for root partition
   - remainder for home partition
 - format root and home paritions to btrfs; format esp to vfat
 - mount partitions to /mnt, /mnt/home, and /mnt/boot
- Installation
   - update `/etc/pacman.d/mirrorlist`
   - `pacstrap /mnt base linux linux-firmware`
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

- for temporary wired connection, run `dhcpcd`
- NetworkManager
  - `pacman -S networkmanager`
  - `systemctl enable NetworkManager`
  - `nmcli device wifi connect <SSID> password <PASSWORD>`
- for wireless adapter that requiers out-of-tree driver,
  - /etc/modules-load.d/
  - /etc/modprobe.d/blacklist.conf

## X11

- `pacman -S xorg xorg-xinit xterm`
- `pacman -S i3`

## Development

- `pacman -S base-devel`
- `pacman -S git meson cmake`

## Wayland
