Cadmium
=======

## Manual Steps

- cadmium builds a bootable usb disk that can install a linux distro on the
  chromebook
- it can be done manually
- prepare disk image
  - `fallocate -l 3GiB my.img`
  - `fdisk my.img`
    - two 64MB partitions of type `ChromeOS kernel`
    - 1 partition of type `Linux filesystem`
  - `cgpt add -i 1 -S 1 -T 2 -P 10 my.img`
  - `cgpt add -i 2 -S 1 -T 2 -P 5 my.img`
  - `sudo losetup -f -P my.img`
- prepare root
  - `mkfs.f2fs /dev/loop0p3`
  - `mount /dev/loop0p3 /mnt`
  - `bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt`
    - untar archlinux bootstrap tarball if x86
  - `mount -t proc none /mnt/proc`
  - `chroot /mnt`
  - `rm /etc/resolv.conf`
  - `echo "nameserver 8.8.8.8" > /etc/resolv.conf`
  - `pacman-key --init`
  - `pacman-key --populate archlinuxarm`
    - `pacman-key --populate archlinux` if x86
  - `pacman -R linux-aarch64`
  - `pacman -Syu`
  - `pacman -S vim vboot-utils f2fs-tools linux-firmware-qcom networkmanager rmtfs-git`
  - `pacman -Scc`
  - `systemctl enable NetworkManager rmtfs`
  - `userdel -r alarm`
    - or keep it and the password is `alarm`
  - `passwd -d root`
    - or keep it and the password is `root`
- prepare kernel
  - build kernel normally
  - pack the kernel with `vbutil_kernel`
  - `cp Image.vboot /dev/loop0p1`
  - `make INSTALL_MOD_PATH=/mnt modules_install`
    - in the chroot, `depmod -a <version`
- clean up and flash
  - `umount /mnt/proc`
  - `umount /mnt`
  - `losetup -D`
  - `cp my.img /dev/sda`
  - `sync`
- installation
  - boot dut from the usb
  - partition the dut disk similar to partitioning the disk image
  - `cp /dev/sda1 /dev/mmcblk0p1`
    - `/dev/nvme0n1p1` if nvme
    - this does not work if the kernel cmdline has hardcoded `root=`
  - `cp /dev/sda3 /dev/mmcblk0p3`
  - `resize.f2fs /dev/mmcblk0p3`
  - `reboot`

## Cadmium Dependencies

- to create disk image
  - `parted`
  - AUR `vboot-utils`
    - uprev to the latest branch (e.g., 109.15236)
    - no `trousers`
    - append `SDK_BUILD=1 USE_FLASHROM=0` to all make invocations
- to download and build kernel,
  - `curl patch aarch64-linux-gnu-gcc`
- to boostrap root,
  - `f2fs-tools qemu-user-static arch-install-scripts wget`
- to package kernel,
  - `uboot-tools dtc`

## Cadmium `build-all`

- the script builds a bootable usb image
  - first partition is a vboot kernel image
  - second partition is unused (for alternative kernel image)
  - last partition is a minimal root with a `/install` script to install
    Cadmium to the internal storage on chromebook
  - use 3G disk image: the minimal root alone is 2.4G
- source `config`
  - `FLAV=arm64-chromebook`
  - `ROOTFS=arch`
  - `FILESYSTEM=f2fs`
  - `KERNEL=kernelorg`
- source `flavor/arm64-chromebook`
  - `ARCH=arm64`
  - `ARCH_UNAME=aarch64`
  - `ARCH_ALARM=aarch64`
  - `BOOTFW=depthcharge`
- source `bootfw/depthcharge/prepare_parts`
  - create an image and set up a loop device
  - use `parted` to `mklabel`
  - use `cgpt` to create 3 partitions
    - `KERNPART=/dev/loop0p1`, kernel, 32M
    - `KERNPART_B=/dev/loop0p2`, kernel, 32M
    - `ROOTPART=/dev/loop0p3`, data, remaining size
  - use `partx` to parse the partition table
- source `kernel/build`
  - workdir is `tmp`
  - download the latest kernel tarball and unpack to `kernel-arm64`
  - apply `kernel/patches/*.patch`
  - use `kernel/config.arm64` as the config
  - build
- mount chroot
  - `mkfs.fsf2 /dev/loop0p3`
  - `mount /dev/loop0p3 tmp/root`
- source `fs/build`
  - copy `qemu-aarch64-static` to chroot `/`
  - use `arch-root` to chroot
  - source `fs/arch/build`
    - download and unpack
      <http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz> to
      chroot
    - `pacman-key --init`
    - `pacman-key --populate archlinuxarm`
    - `echo "nameserver 1.1.1.1" > /etc/resolv.conf`
    - `pacman -Syu --noconfirm networkmanager`
      - olv: add `pacman -R --noconfirm linux-aarch64` before the line to save
        time
    - `systemctl enable --now NetworkManager.service`
  - source `fs/arch/info`
    - `FS_INST_PKG="pacman -Syu --noconfirm"`
      - olv: add `--needed` to save time
    - `FS_PKGS_CD_BASE="some essential packages"`
    - many others
  - install `$FS_PKGS_CD_BASE` to chroot
  - copy `fs/firmware/*` to `/lib/firmware`
  - create `/CdFiles`
    - copy `fs`, `board`, `baseboard` to `/CdFiles`
    - copy `fs/install` to `/root` (home directory of root)
    - generate `/CdFiles/config`
    - generate `/lib/firmware/rmtfs/modem_*`
    - git clone `qmic`, `qrtr`, and `rmtfs` to `/CdFiles`
    - `passwd -d root`
- source `bootfw/depthcharge/package`
  - `mkimage` a packed kernel image
    - `lz4` kernel image
    - `kernel/arm64.kernel.its` is the dtb to pack
  - use `vbutil_kernel` to generate `vmlinux.kpart`, `vmlinux.kpart.p2`, and
    `oxide.kpart`
    - only differ in cmdlines
    - use `kernel/cmdline`,`kernel/cmdline.p2`, and `kernel/cmdline.oxide`
      respectively
- dd `vmlinux.kpart` to `/dev/loop0p1`
  - `vmlinux.kpart.p2` is not used
- copy `oxide.kpart` to chroot
- `make -C tmp/linux-arm64 modules_install` to chroot
- `losetup -d /dev/loop0`

## Cadmium `/root/install`

- `INSTMED="emmc"`
- `CADMIUMROOT="/CdFiles"`
- `TARGET` is set to one of the boards under `board`
  - it uses `/sys/firmware/devicetree/base/compatible` to auto-detect
- source `/CdFiles/board/$TARGET/boardinfo`
  - use `coachz` as an example
  - `BASEBOARD=trogdor`
  - `BOARD=coachz`
  - `TYPE=convertible-laptop`
  - `WLR_DISP=eDP-1`
  - `DISP_ROTATION=0`
- source `/CdFiles/baseboard/$BASEBORD/boardinfo`
  - use `trogdor` as an example
  - `EMMC_OFFSET=0`
  - `SOC=sc7180`
- detect battery level and bail on low battery
- wifi
  - build and install `qmic`, `qrtr`, `rmtfs`
  - `systemctl start rmtfs` for `ath10k_snoc` wifi
  - `nmcli` and `nmtui` to connect to wifi
- `MMCDEV=/dev/mmcblk0`
- source `/CdFiles/config`
- source `/CdFiles/fs/arch/info`
  - `FS_PKGS_CD_BOOTFW_DEPTHCHARGE="uboot-tools vboot-utils"`
- install `$FS_PKGS_CD_BOOTFW_DEPTHCHARGE`
- install to `/dev/mmcblk0`
  - partition
  - copy kernel
  - mkfs
  - source `/CdFiles/fs/build`
  - copy `/CdFiles`
  - copy `/lib/firmware`
  - copy kernel modules
  - build and install `qmic`, `qrtr`, `rmtfs`
  - install `$FS_PKGS_CD_BOOTFW_DEPTHCHARGE`
  - set root passwd
  - install fw, base, and ui packages
- done

