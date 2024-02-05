SYSLINUX
========

## Overview

- <https://wiki.syslinux.org/wiki/index.php?title=The_Syslinux_Project>
- the project consists of 5 bootloaders
  - `SYSLINUX` boots from FAT
  - `ISOLINUX` boots from ISO 9660
  - `PXELINUX` boots from PXE
  - `EXTLINUX` boots from ext2/ext3/ext4/btrfs/fat/ntfs/ufs/ufs2/xfs
  - `MEMDISK` is chainloaded by other bootlaoders and can emulate a disk image
    as a real disk
    - this is to boot, for example, dos from a floppy disk image
- `SYSLINUX` is a boot loader for the Linux operating system which runs on an
  MS-DOS/Windows FAT filesystem. It is intended to simplify first-time
  installation of Linux, and for creation of rescue and other special purpose
  boot disks.
- `ISOLINUX` is a boot loader for Linux/i386 that operates off ISO 9660/El
  Torito CD-ROMs in "no emulation" mode. This avoids the need to create an
  "emulation disk image" with limited space (for "floppy emulation") or
  compatibility problems (for "hard disk emulation").
  - it also provides `isohybrid` to create a hybrid iso image that can be
    flashed to USB
  - this is how most distributions master their installation iso images
- `PXELINUX` is a Syslinux derivative, for booting from a network server using
  a network ROM conforming to the Intel PXE (Pre-Execution Environment)
  specification
- `EXTLINUX` is a Syslinux variant which boots from a Linux filesystem.
