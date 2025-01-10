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
    - 1M for BIOS boot partition (type 4), if bios
    - 1G for EFI system partition (type 1), esp
      - `mkfs.vfat -F32 <part>`
    - remainder for root partition
      - `mkfs.ext4 <part>`
  - mount partitions to `/mnt` and `/mnt/boot`
- Bootstrap
  - update `/etc/pacman.d/mirrorlist`
    - `Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch`
    - `Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch`
  - `pacstrap /mnt base`
  - `genfstab -U /mnt >> /mnt/etc/fstab` and double-check
- Enter chroot
  - `arch-chroot /mnt`
  - `hwclock --systohc`
    - this updates `/etc/adjtime` so should be done in chroot
  - install more packages
    - `pacman -S linux linux-firmware dosfstools sudo vim`
    - `pacman -S dhcpcd iwd wpa_supplicant`, at most one of them should suffice
    - `pacman -S grub`, if using BIOS
    - `pacman -S linux-headers broadcom-wl-dkms`, or other out-of-tree drivers
  - uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen` and run `locale-gen`
  - `systemd-firstboot --prompt`, or
    - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
    - `echo LANG=en_US.UTF-8 > /etc/locale.conf`
    - `echo <host-name> > /etc/hostname`
    - `passwd` to set a password for root
  - create user
    - `useradd -m -G wheel <user>`
    - `passwd <user>`
    - `visudo` to uncomment `%wheel ALL=(ALL:ALL) NOPASSWD: ALL`
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

- network
  - for quick connection, see `Installation`
  - `echo -e '[Match]\nName=*\n[Network]\nDHCP=yes' > /etc/systemd/network/all.network`
  - `systemctl enable --now iwd systemd-networkd systemd-resolved`
  - `iwctl station <iface> connect <ssid>`
  - for wireless adapter that requiers out-of-tree driver,
    - `dkms autoinstall`
    - `/etc/modules-load.d`
    - `/etc/modprobe.d/blacklist.conf`
- login as user
  - `git clone https://github.com/olvaffe/olv-etc.git`
  - `./olv-etc/create-links`
- packages
  - base
    - `base linux linux-firmware intel-ucode`
    - `dosfstools btrfs-progs`
    - `zram-generator`
      - `echo -e '[zram0]\nzram-size = ram' > /etc/systemd/zram-generator.conf`
      - `systemctl daemon-reload`
      - `zramctl`
    - `sudo vim`
    - `openssh wireguard-tools`
  - network
    - `iwd` or `wpa_supplicant`
    - `networkmanager`
    - `linux-headers broadcom-wl-dkms`
  - tools
    - `bc imagemagick unzip zip wget`
    - `dmidecode usbutils`
    - `htop iw lsof`
    - `man-db man-pages`
    - `picocom`
  - devel
    - `base-devel ccache ctags gdb git meson cmake`
    - `perf trace-cmd strace debuginfod`
    - `llvm clang`
    - `python-pip ruff`
  - mesa devel
    - `python-mako python-yaml wayland-protocols libxrandr`
    - `llvm glslang libclc spirv-llvm-translator`
    - `vulkan-validation-layers vulkan-extra-layers`
  - cross-compile
    - `multilib-devel`
      - uncomment the `[multilib]` section in `/etc/pacman.conf`
    - `aarch64-linux-gnu-gcc`
    - `qemu-user-static qemu-user-static-binfmt`
  - gui
    - `sway polkit i3status swayidle swaylock mako`
    - `mesa mesa-utils vulkan-intel vulkan-radeon vulkan-tools`
    - `noto-fonts noto-fonts-cjk`
    - `alacritty google-chrome gtk4`
    - `brightnessctl wl-clipboard wayland-utils`
    - `xorg-xwayland`
    - `fcitx5-chewing fcitx5-configtool fcitx5-gtk`
    - `imv grim slurp`
    - `mpv intel-media-driver`
  - legacy x11
    - `xorg xorg-xinit i3 xterm`
  - audio
    - `pipewire pipewire-pulse`
    - `pavucontrol`
  - bluetooth
    - `bluez bluez-utils`
    - `bcm20702a1-firmware`
  - printer
    - `cups`
    - `cnrdrvcups-lb samsung-unified-driver-printer`
  - 32-bit
    - `lib32-{mesa,libdrm,libunwind,libx11}`

## Installation on USB drive

- <https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium>
- bootstrap a disk image
  - `fallocate -l 4G arch.img`
  - `fdisk arch.img`
    - esp, cros kernel a/b, arch
    - label esp and arch
  - `sudo losetup -f -P arch.img`
  - `sudo mkfs.vfat -F32 /dev/loop0p1`
  - `sudo mkfs.f2fs -O extra_attr,compression /dev/loop0p4`
  - `mkdir arch`
  - `sudo mount /dev/loop0p4 arch`
  - `sudo tar -xf archlinux-bootstrap-x86_64.tar.gz -C arch --strip 1`
  - `sudo rm arch/pkglist.x86_64.txt arch/version`
  - `sudo vim arch/etc/pacman.d/mirrorlist`
  - `sudo umount arch`
  - `rmdir arch`
  - `losetup -D`
- install packages
  - `sudo systemd-nspawn -i arch.img --private-users=identity`
    - it is smart enough to mount esp and root automatically
  - `pacman-key --init`
  - `pacman-key --populate`
  - `pacman -R arch-install-scripts`
  - `pacman -Syu linux linux-firmware dosfstools f2fs-tools zram-generator \
                 iwd sudo vim man-db base-devel git sway polkit i3status \
                 swayidle swaylock mako alacritty noto-fonts wayland-utils \
                 mesa mesa-utils vulkan-tools`
  - `pacman -Scc`
- setup
  - `passwd`
  - `vim arch/etc/fstab`
  - `vim /etc/locale.gen`
  - `locale-gen`
  - `echo LANG=en_US.UTF-8 > /etc/locale.conf`
  - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
  - `echo usb > /etc/hostname`
  - `echo -e '[zram0]\nzram-size = ram' > /etc/systemd/zram-generator.conf`
- create a normal user
  - `useradd -m -G wheel olv`
  - `passwd olv`
  - `visudo`
    - allow wheel to sudo
  - `su - olv`
  - `git clone --recurse-submodules https://github.com/olvaffe/olv-etc.git`
  - `rm .bashrc .bash_profile`
  - `./olv-etc/create-links`
  - `exit`
- install bootloader
  - `bootctl install`
  - `echo 'default arch.conf' >> /boot/loader/loader.conf`
  - `vim /boot/loader/entries/arch.conf`
- update
  - `pacman -Syu`
  - `pacman -Scc`
- boot with nspawn
  - THIS DOES NOT WORK
    - the more common way is to bind-mount wayland/x11/pipewire sockets
  - `sudo systemd-nspawn -i arch.img --private-users=identity -b --bind /dev/dri --bind /dev/input --bind /dev/snd`
    - man `systemd.resource-control`
      - `systemctl set-property machine-arch.img.scope DeviceAllow='char-drm rw'`
      - also `char-alsa` and `char-input`
    - fix up `/etc/groups` in the chroot
    - `--network-veth`
    - <https://systemd.io/CONTAINER_INTERFACE/>
    - sway, libinput, pipewire rely on udev for device discovery
    - systemd-udev is not running

## Installation in crosvm

- prepare iso, kernel and initramfs
  - download iso
  - extract kernel and initramfs under `arch/boot/x86_64` in iso
    - use loop mount or 7z
- start crosvm
  - `--rwdisk arch.img --disk <path-to-iso> -p archisodevice=/dev/vdb`
  - `--host_ip 192.168.0.1 --netmask 255.255.255.0 --mac 12:34:56:78:9a:bc`
  - `--disable-sandbox`
- installation
  - for network,
    - `ip addr add 192.168.0.2/24 dev enp0s5`
    - `ip route add default via 192.168.0.1`
  - `arch.img` is `/dev/vda`
    - `/boot`, `linux`, `linux-firmware`, and bootloader not needed
    - but can still partition, format, and bootstrap normally for qemu

## Tidy Up an Existing Installation

- `pacman -Qeq` to get explicitly packages
- `pacman -D --asdeps` to mark them deps
- `pacman -D --asexplicit` to mark desired packages explicit
- `pacman -Qtdq` to get packages to remove
- to find orphaned files,
  - `pacman -Ql $(pacman -Qq) | cut -d' ' -f2- | sort | uniq > pacman.list`
  - `find / -xdev -path "/home/*" -prune -o -print | sort > find.list`

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

- archlinux-keyring
- bash
- bzip2
- coreutils
  - ls, mv cp, sync, cat, chmod, chown, basename, cut, date, df, du, echo,
    env, chroot, etc.
- file
- filesystem
  - directory strucutre (e.g., /root, /usr/bin)
  - basic links (e.g., /bin to /usr/bin)
  - some files in /etc (e.g., passwd, group, mtod)
- findutils
  - find, xargs
- gawk
- gcc-libs
  - gcc runtime libraries (`libgcc_s.so`, `libstdc++.so`, `libatomic.so`,
    `libasan.so`)
- gettext
- glibc
  - headers
  - libraries (ld-linux-x86-64, libc, libdl, libm, libresolv, crt\*.o, etc)
  - tools (ldconfig, ldd, locale, locale-gen, iconv, etc)
- grep
- gzip
- iproute2
  - ip, ss, bridge
- iputils
  - ping, tftpd, etc.
- licenses
- pacman
- pciutils
  - lspci
- procps-ng
  - ps, top, free, pidof, pkill
- psmisc
  - fuser, killall, pstree, etc.
- sed
- shadow
  - grouadd, passwd, useradd, lastlog, newgidmap, newuidmap, etc.
- systemd
  - systemd, systemctl, timedatectl, networkctl, resolvect, loginctl,
    localectl, udevadm, etc.
- systemd-sysvcompat
  - init (symlink to systemd), reboot, shutdown, poweroff, halt (symlinks to
    systemctl)
- tar
- util-linux
  - agetty, blkid, dmesg, fdisk, kill, login, fsck, mkfs, mount, mountpoint,
    findmnt, rfkill, su, switch_root, nsenter, unshare, lsns, etc.
- xz

## Big Picture Mode

- `plymouth`
  - `pacman -S plymouth`
  - use `plymouth-set-default-theme` to set/get the default theme or list all
    available themes
- initramfs
  - edit `/etc/mkinitcpio.conf` to add `plymouth` before `autodetect` hook
  - `mkinitcpio -p linux` to regenerate initramfs
- bootloader
  - edit `/boot/loader/entries/arch.conf`
  - add `quiet` such that kernel does not print messages before `plymouth`
    takes over
  - add `splash` such that `plymouth` shows the theme
- kernel
  - i915 has a "fastboot" mode that is enabled by default since skylake and
    can be forced elsewhere with `i915.fastboot=1`
  - amdgpu has a "seamless" mode that is only enabled on DCN3+ (RDNA2+) APU
- systemd
  - add `/etc/systemd/system/getty@tty1.service.d/autologin.conf` to override
    `getty@tty1.service`
  - `[Service]`
  - `ExecStart=`
  - `ExecStart=-/sbin/agetty -o '-p -f -- olv' --noclear --noissue --skip-login %I $TERM`
- login
  - `touch ~/.hushlogin` to do a quiet login

## mkinitcpio

- options
  - `-g <output>` writes the initramfs to `<output>`; dry run if not specified
  - `-k <ver>` specifies the kernel version; defult to `uname -r`
  - `-v` be verbose
  - `-L` lists hooks
  - `-H <hook>` shows the help message of `<hook>`
  - `-p <present>` creates an initramfs according to
    `/etc/mkinitcpio.d/<present>.preset`
- `/etc/mkinitcpio.conf` by default enables these hooks in order
  - `base` adds `busybox`, `kmod`, `/init`, and others
  - `udev` adds `systemd-udevd`, `udevadm`, and others
  - `autodetect` scans `/sys` for `modalias`s and invokes `modprobe -R` to map
    them to module names cached in `_autodetect_cache` variable; hooks after
    `autodetect` only consider modules in `_autodetect_cache`
  - `modconf` adds `modprobe` configuration files
  - `kms` adds modules related to drm as well as their dependencies/firmwares
  - `keyboard` adds modules related to input
  - `keymap` adds the keymap if any
  - `consolefont` adds the console font if any
  - `block` adds modules related to block
  - `filesystems` adds modules related to fs
  - `fsck` adds fsck for different filesystems
- `plymouth` hook
  - it adds `plymouth`, the current theme (returned by
    `plymouth-set-default-theme`), and the runscript
