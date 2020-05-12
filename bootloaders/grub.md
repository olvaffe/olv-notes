GRUB
====

## bootloader

- MBR
  - the first 512 bytes of a disk
  - After BIOS, it jumps to MBR
  - also holding the partition table
    - A partition can be marked bootable (active).  It is used by boot
      loaders from old days to determine which partition to boot.  Not used by
      grub.
- PBR (partition boot sector)
  - the first 512 bytes of a partition
  - boot loaders can also be installed to PBR, suppose the one installed to
    MBR knows PBR
- grub
  - `grub-install` is a script to install grub to the disk
    - For EFI, it creates `/boot/efi/EFI/debian`
    - it creates `/boot/grub` and the device map using `grub-mkdevicemap`
    - it copies grub binaries to `/boot/grub`
    - it invokes `grub-mkimage` to create the core image
    - For EFI, it invokes `efibootmgr` to copy the image to
      `/boot/efi/EFI/debian` and boot from it
    - For non-EFI, it invokes `grub-setup` to install a boot image to MBR/PBR
      - whether grub (boot.img) is installed in MBR or PBR, it loads another fixed
        sector which holds the first 512 bytes of core.img.  It then jumps to it and
        loads the rest of core.img
      - core.img has the ability to load modules and use filesystems
