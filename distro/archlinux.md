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
    - wireless: `iwctl station <iface> connect <ssid>`
      - no easy way if the wireless adapter is not supported by the live image
  - wait until system clock synced
    - `timedatectl`
  - partition disk with fdisk
    - 1M for BIOS boot partition (type 4), if bios
    - 1G for EFI system partition (type 1), esp
      - `mkfs.fat -F32 <part>`
    - remainder for root partition
      - `mkfs.ext4 <part>`
  - mount partitions to `/mnt` and `/mnt/boot`
- Bootstrap
  - update `/etc/pacman.d/mirrorlist` if desired
    - `Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch`
    - `Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch`
  - `pacstrap -K /mnt base`
  - `genfstab -U /mnt >> /mnt/etc/fstab` and double-check
- Enter chroot
  - `arch-chroot /mnt`
  - `hwclock --systohc`
    - this updates `/etc/adjtime` so should be done in chroot
  - install more packages
    - `pacman -S linux linux-firmware intel-ucode dosfstools btrfs-progs`
    - `pacman -S sudo vim`
    - `pacman -S zram-generator`
      - `echo '[zram0]' > /etc/systemd/zram-generator.conf`
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
  - install bootloader
    - `bootctl install` for EFI
    - create `/boot/loader/entries/arch.conf`
      - `title	Arch Linux`
      - `linux	/vmlinuz-linux`
      - `initrd	/initramfs-linux.img`
      - `options root=... loglevel=7`
    - or, on a BIOS system,
      - `grub-install <dev>`
      - `grub-mkconfig -o /boot/grub/grub.cfg`
- Reboot

## Post-Installation

- network
  - for quick connection, see `Installation`
  - `echo -e '[Match]\nType=wlan\n[Network]\nDHCP=yes' > /etc/systemd/network/wlan.network`
  - `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`
  - `systemctl enable --now iwd systemd-networkd systemd-resolved`
  - `iwctl station <iface> connect <ssid>`
  - for wireless adapter that requiers out-of-tree driver,
    - `dkms autoinstall`
    - `/etc/modules-load.d`
    - `/etc/modprobe.d/blacklist.conf`
  - `timedatectl set-ntp true`
- login as user
  - `git clone --recurse-submodules https://github.com/olvaffe/olv-etc.git`
  - `./olv-etc/create-links`
- packages
  - base
    - `base linux linux-firmware intel-ucode`
    - `dosfstools btrfs-progs`
    - `sudo vim`
    - `zram-generator`
  - network
    - `openssh wireguard-tools`
    - `iwd` or `wpa_supplicant`
    - `networkmanager`
    - `linux-headers broadcom-wl-dkms`
  - tools
    - `bc unzip zip`
    - `dmidecode usbutils hdparm iw`
    - `lsof htop iotop`
    - `rsync wget`
    - `man-db man-pages`
    - `picocom`
  - devel
    - `base-devel ccache ctags gdb git meson cmake`
    - `perf trace-cmd strace debuginfod`
    - `llvm clang`
    - `rustup`
  - mesa devel
    - `wayland-protocols libxrandr`
    - `llvm glslang libclc spirv-llvm-translator`
    - `vulkan-validation-layers vulkan-extra-layers`
    - `python -m venv ~/.pip`
      - `pip install packaging mako pyyaml`
  - cross-compile
    - `aarch64-linux-gnu-gcc`
    - `qemu-user-static qemu-user-static-binfmt`
  - gui
    - `sway polkit i3status swayidle swaylock mako`
    - `mesa mesa-utils vulkan-tools vulkan-intel vulkan-radeon`
    - `noto-fonts noto-fonts-cjk noto-fonts-emoji`
    - `brightnessctl wl-clipboard wayland-utils`
    - `fcitx5-chewing fcitx5-configtool fcitx5-gtk`
    - `xdg-desktop-portal-gtk xdg-desktop-portal-wlr`
    - `alacritty google-chrome gtk4`
    - `swayimg grim slurp`
    - `mpv intel-media-driver`
    - `xorg-xwayland`
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
    - uncomment the `[multilib]` section in `/etc/pacman.conf`
    - `lib32-{mesa,libdrm,libunwind,libx11}`

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

## Disk Image

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
  - `echo -e '[zram0]' > /etc/systemd/zram-generator.conf`
- create a normal user
  - `useradd -m -G wheel olv`
  - `passwd olv`
  - `visudo`
    - allow wheel to sudo
- install bootloader
  - `bootctl install`
  - `echo 'default arch.conf' >> /boot/loader/loader.conf`
  - `vim /boot/loader/entries/arch.conf`
- update
  - `pacman -Syu`
  - `pacman -Scc`

## Tidy Up an Existing Installation

- `pacman -Qeq` to get explicitly packages
- `pacman -D --asdeps` to mark them deps
- `pacman -D --asexplicit` to mark desired packages explicit
- `pacman -Qttdq` to get packages to remove
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

## `base` package

- archlinux-keyring
- bash
- bzip2
- coreutils (`ls`, `mv`, `echo`, `env`, etc.)
- file
- filesystem
  - directory strucutre (e.g., /root, /usr/bin)
  - basic links (e.g., /bin to /usr/bin)
  - some files in /etc (e.g., passwd, group, mtod)
- findutils (`find`, `xargs`, etc.)
- gawk
- gcc-libs (`libgcc_s.so`, `libstdc++.so`, `libatomic.so`, etc.)
- gettext
- glibc (C headers and runtime, loader, `ldconfig`, `ldd`, `locale-gen`, etc.)
- grep
- gzip
- iproute2 (`ip`, `ss`, etc.)
- iputils (`ping`, etc.)
- licenses
- pacman
- pciutils (`lspci`, etc.)
- procps-ng (`ps`, `top`, `uptime`, `free`, `kill`, etc.)
- psmisc (`killall`, `pstree`, `fuser`, etc.)
- sed
- shadow (`useradd`, `groupadd`, `passwd`, etc.)
- systemd (`systemctl`, `networkctl`, `udevadm`, etc.)
- systemd-sysvcompat (`init`, `reboot`, `shutdown`, etc.; all symlinks)
- tar
- util-linux (`agetty`, `login`, `dmesg`, `mkfs`, `mount`, etc.)
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

## Kernel

- `linux` installs to `/usr/lib/modules/<version>` entirely
- `kmod` includes `/usr/share/libalpm/hooks/60-depmod.hook`
  - it runs `/usr/share/libalpm/scripts/depmod` when there are changes to
    `/usr/lib/modules/*`
  - the script runs `depmod`
- `mkinitcpio` includes `/usr/share/libalpm/hooks/60-mkinitcpio-remove.hook`
  and `/usr/share/libalpm/hooks/90-mkinitcpio-install.hook`
  - they run `/usr/share/libalpm/scripts/mkinitcpio` when there are relevant
    changes
  - on install, the script
    - creates `/etc/mkinitcpio.d/linux.preset` from
      `/usr/share/mkinitcpio/hook.preset` if missing
    - copies `/usr/lib/modules/<version>/vmlinuz` to `/boot/vmlinuz-linux`
    - runs `mkinitcpio -p linux`

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
