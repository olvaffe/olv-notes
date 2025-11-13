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
  - `timedatectl` and wait until system clock synced
  - partition and format disk
    - `fdisk <dev>`
      - 2GB for EFI system partition (ESP)
      - remainder for root partition
    - `mkfs.fat -F32 <part>`
    - `mkfs.ext4 <part>`
    - if luks,
      - `cryptsetup luksFormat <part>`
      - `systemd-cryptsetup attach root <part>`
    - if btrfs,
      - `mkfs.btrfs <part>`
      - `mount <part> /mnt`
      - `btrfs subvolume create /mnt/@`
      - `btrfs subvolume create /mnt/@home`
      - `btrfs subvolume create /mnt/@srv`
      - `btrfs subvolume set-default /mnt/@`
      - `umount /mnt`
  - mount partitions to `/mnt` and `/mnt/boot`
    - if btrfs,
      - `mount -o compress=zstd <part> /mnt`
      - `mkdir /mnt/{boot,home,srv}`
      - `mount <esp> /mnt/boot`
      - `mount -o subvol=@home <part> /mnt/home`
      - `mount -o subvol=@srv <part> /mnt/srv`
- Bootstrap
  - update `/etc/pacman.d/mirrorlist` if desired
    - `Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch`
    - `Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch`
  - `pacstrap -K /mnt base`
  - `genfstab -U /mnt >> /mnt/etc/fstab` and edit
    - replace expanded default options by `defaults`
- Configure
  - `arch-chroot /mnt`
  - `hwclock --systohc`
    - this updates `/etc/adjtime` so should be done in chroot
  - install more packages
    - `pacman -S sudo vim dosfstools btrfs-progs`
    - `pacman -S iwd wpa_supplicant`, either suffices
    - `pacman -S {intel,amd}-ucode linux-firmware-{amdgpu,intel,mediatek}`
    - `pacman -S linux systemd-ukify`
    - `pacman -S linux-headers broadcom-wl-dkms`, or other out-of-tree drivers
  - generate locale
    - uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen`
    - `locale-gen`
  - `systemd-firstboot --prompt --force`, or
    - `echo LANG=en_US.UTF-8 > /etc/locale.conf`
    - `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
    - `echo <host-name> > /etc/hostname`
    - `passwd` to set a password for root
  - create user
    - `useradd -m -G wheel <user>`
    - `passwd <user>`
    - `visudo` to uncomment `%wheel ALL=(ALL:ALL) NOPASSWD: ALL`
  - `bootctl install`
  - generate uki
    - create `/etc/kernel/cmdline`
      - `root=<part> loglevel=7`
      - if luks, `rd.luks.name=<LUKS_UUID>=root root=/dev/mapper/root`
        - `cryptsetup luksUUID <part>` to get uuid
    - edit `/etc/mkinitcpio.d/linux.preset`
      - comment out `default_image`
      - `default_uki=/boot/EFI/Linux/arch-linux.efi`
    - if luks, edit `/etc/mkinitcpio.conf`
      - add `sd-encrypt` after `sd-vconsole` in `HOOKS`
    - `mkinitcpio -p linux`
    - `rm /boot/initramfs-linux.img`
- Reboot

## Post-Installation

- network
  - for quick connection, see `Installation`
  - `echo -e '[Match]\nType=wlan\n[Network]\nDHCP=yes' > /etc/systemd/network/60-wlan.network`
  - `systemctl enable --now iwd systemd-networkd systemd-resolved`
  - `iwctl station <iface> connect <ssid>`
  - for wireless adapter that requiers out-of-tree driver,
    - `dkms autoinstall`
    - `/etc/modules-load.d`
    - `/etc/modprobe.d/blacklist.conf`
  - `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`
  - `timedatectl set-ntp yes`
- secure boot
  - `pacman -S sbctl`
  - `sbctl create-keys`
  - if preserving oem keys,
    - `sbctl export-enrolled-keys --dir /var/lib/sbctl/keys/custom --disable-landlock`
    - `mv /var/lib/sbctl/keys/custom/{DB,db}`
  - reboot and clear oem keys to enter setup mode
  - `sbctl enroll-keys`
    - `-c` if preserving oem keys
  - `sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI`
  - `sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi`
  - `sbctl sign -s /boot/EFI/Linux/arch-linux.efi`
- luks
  - create `/etc/kernel/uki.conf`
    - `[PCRSignature:initrd]`
    - `Phases=enter-initrd`
    - `PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key.pem`
    - `PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem`
  - `ukify genkey -c /etc/kernel/uki.conf` generates the key to sign pcr
    policies
  - `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 --tpm2-public-key-pcrs=11 --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem <part>`
  - `mkinitcpio -p linux`
- login as user
  - `git clone --recurse-submodules https://github.com/olvaffe/olv-etc.git`
  - `./olv-etc/create-links`
- packages
  - `zram-generator`
    - `echo '[zram0]' > /etc/systemd/zram-generator.conf`
  - `steam`
    - uncomment the `[multilib]` section in `/etc/pacman.conf` first
  - media: `intel-media-driver`
  - legacy x11: `xorg xorg-xinit i3 xterm`
  - bluetooth: `bluez bluez-utils`
  - printer: `cups cnrdrvcups-lb samsung-unified-driver-printer`

## Tidy Up an Existing Installation

- `pacman -Qeq` to get explicitly packages
- `pacman -D --asdeps` to mark them deps
- `pacman -D --asexplicit` to mark desired packages explicit
- `pacman -Qttdq` to get packages to remove
- to find orphaned files,
  - `pacman -Ql $(pacman -Qq) | cut -d' ' -f2- | sort | uniq > pacman.list`
  - `find / -xdev -path "/home/*" -prune -o -print | sort > find.list`

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
- custom uki image
  - `mkinitcpio -k <version> --kernelimage <bzImage> -U /boot/EFI/Linux/custom.efi`
    - ignore errors on missing modules if they are unnecessary or built-in
    - the kernel image must have efi stub

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
  - `base` adds `busybox`, `kmod`, `blkid`, `mount`, `/init`, and others
  - `systemd` adds systemd and units
    - it can replace `base` and `udev`
  - `autodetect` scans `/sys` for `modalias`s and invokes `modprobe -R` to map
    them to module names cached in `_autodetect_cache` variable; hooks after
    `autodetect` only consider modules in `_autodetect_cache`
  - `microcode` adds cpu ucode
  - `modconf` adds `modprobe` configuration files
  - `kms` adds modules related to drm as well as their dependencies/firmwares
  - `keyboard` adds modules related to input
  - `keymap` adds the keymap if any
  - `sd-vconsole` adds the keymap if any
    - it can replace `keymap` and `consolefont`
  - `block` adds modules related to block
  - `filesystems` adds modules related to fs
  - `fsck` adds fsck for different filesystems
- `sd-encrypt` hook
  - it adds dm, crypto, and tpm modules
  - it adds udev rules, systemd crypt units, etc.
- `plymouth` hook
  - it adds `plymouth`, the current theme (returned by
    `plymouth-set-default-theme`), and the runscript

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
