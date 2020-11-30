Kernel Config (Raspberry Pi)
============================

## Prepare

- install cross compiler
  - `crossbuild-essential-arm64` on Debian-like
  - `aarch64-linux-gnu-gcc` on Arch-like
- use these variables
  - `O=RPI`
  - `ARCH=arm64`
  - `CROSS_COMPILE=aarch64-linux-gnu-`
- save this as `.config`
  <https://raw.githubusercontent.com/raspberrypi/linux/rpi-5.10.y/arch/arm64/configs/bcm2711_defconfig>
- `make olddefconfig`
- `make menuconfig`

## Config

- select `General setup`
  - deselect `Local version - append to kernel release`
  - select `Preemption Model (Voluntary Kernel Preemption (Desktop))`
  - update `Initramfs source file(s)` if diskless
- select `Boot Options`
  - update `Default kernel command string`
- select `General architecture-dependent options`
  - select `Provide system calls for 32-bit time_t`
- select `Networking support`
  - deselect `Amateur Radio support`
  - deselect `CAN bus subsystem support`
  - deselect `WiMAX Wireless Broadband support`
  - deselect `Plan 9 Resource Sharing Support (9P2000)`
  - deselect `NFC subsystem support`
- select `File systems`
  - deselect `The Extended 4 (ext4) filesystem`
  - deselect `Reiserfs support`
  - deselect `JFS filesystem support`
  - deselect `XFS filesystem support`
  - deselect `GFS2 file system support`
  - deselect `OCFS2 file system support`
  - deselect `NILFS2 file system support`
  - deselect `F2FS filesystem support`
  - deselect `Dnotify support`
  - deselect `Quota support`
  - deselect `Old Kconfig name for Kernel automounter support`
  - deselect `Overlay filesystem support`
  - select `CD-ROM/DVD Filesystems`
    - deselect `ISO 9660 CDROM file system support`
    - deselect `UDF file system support`
  - select `DOS/FAT/EXFAT/NT Filesystems`
    - deselect `MSDOS fs support`
    - select `Enable FAT UTF-8 option by default`
    - deselect `NTFS file system support`
  - select `Pseudo filesystems`
    - select `Tmpfs virtual memory file system support (former shm fs)`
    - select `Tmpfs POSIX Access Control Lists`
    - select `Tmpfs extended attributes`
  - deselect `Miscellaneous filesystems`
  - deselect `Network File Systems`
  - select `Native language support`
    - deselect all but...
    - select `Codepage 437 (United States, Canada)`
    - select `ASCII (United States)`
    - select `NLS ISO 8859-1  (Latin 1; Western European Languages)`
    - select `NLS UTF-8`
  - deselect `Distributed Lock Manager (DLM)`
- select `Kernel hacking`
  - select `Tracers`
    - select `Kernel Function Tracer`
    - select `Trace syscalls`

## Config: Device Drivers

- TODO

## Build

- `make`
- copy `arch/arm64/boot/Image` as `kernel8.img`
- set `arm_64bit=1` in `config.txt`

## Tips

- No DRM?
  - `echo 0x1ff > /sys/module/drm/parameters/debug`
  - `[drm:vc5_hdmi_init_resources [vc4]] ERROR Failed to get HDMI state machine clock`
    - make sure `vc4` is loaded after `raspberrypi-clk`
  - `[drm:vc4_hdmi_bind [vc4]] Failed to get ddc i2c adapter by node
    - make sure `vc4` is loaded after `i2c-brcmstb`
