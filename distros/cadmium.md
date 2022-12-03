Cadmium
=======

## Dependencies

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

## `build-all`

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

## `/root/install`

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

## Qualcomm Modem/WiFi/BT?

- <https://github.com/andersson/qmic.git>
  - a tool to parse `.qmi` into `.c` and `.h`
  - looking at a qmi file, it seems to be for rpc serialization and
    deserialization
- <https://github.com/andersson/qrtr.git>
  - uses `AF_QIPCRTR` socket; according to kernel `CONFIG_QRTR`,
    - Qualcomm IPC router protocol
    - The protocol is used to communicate with services provided by other
      hardware blocks in the system
  - this userspace tool seems to be used for service discovery and communication
  - `qrtr-ns.service` provides QIPCRTR Name Service
- <https://github.com/andersson/rmtfs>
  - depends on qmic to compile `qmi_rmtfs.qmi`
  - depends on libqrtr for service discovery and communication
  - according to kernel `CONFIG_QCOM_RMTFS_MEM`,
    - The Qualcomm remote filesystem memory driver is used for allocating and
      exposing regions of shared memory with remote processors for the purpose
      of exchanging sector-data between the remote filesystem service and its
      clients
  - `rmtfs.service` provides Qualcomm remotefs service
    - it starts rmtfs with
      - `-r`: read only
      - `-P`: use partitions
      - `-s`: sync for remoteproc
- `/lib/firmware/rmtfs/modem_*`
  - there are four 2MB initially-zeroed files: `fs1 fs2 fsg fsc`
  - my guess is the modem needs to read data from the cpu
  - rmtfs uses those the 4 files to provide 4 remote filesystems to the modem
