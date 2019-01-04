UEFI
====

## GPT disk

* <http://www.rodsbooks.com/gdisk/>
* A GPT disk is a disk that uses GUID Partition Table
  * instead of MBR
* kernel should enable advanced partition table
* GPT disk for data
  * OS support is good
* GPT disk for booting
  * Windows requires UEFI (instead of BIOS) to boot from a GPT disk, and only
    64-bit version supports it
  * GRUB2 requires UEFI or a BIOS boot partition to boot from a GPT disk
* Partition Alignment
  * partitions are expressed by 512-byte sectors
  * because physical sector size might be 4096-byte, align partition to 8 sectors
  * traditionally, partitions should align to 63 sectors.  So one might also
    want to align partitions to 8*63 sectors.
* Partitioning for booting
  * UEFI requires ESP, EFI System Partition, to store boot loaders, drivers, and
    other files.  Size should be 200MiB
  * BIOS+GRUB2 requires BIOS Boot Partition to store the second stage
    bootloader.  Size should be 1MiB
  * OSX requires a non-ESP partition to have 128MB unused space after it
  * Windows requires ESP to be the first partition, followed by a MSR partition.
    Size should be 128MiB.
* ESP
  * there should be a bootloader for each OS
    * though some bootloaders can load multiple OS kernels
  * Bootloaders should be in `EFI/` subdirectory
  * Windows bootloader is at `EFI/Microsoft/bootmgfw.efi`
  * EFI provides its own bootloader, which is a boot manager that allows one to
    choose another bootloader
    * rEFIt is a better choice for a boot manager
  * Copy your bootloader to `EFI/Boot/bootx64.efi` to make it the default
  * Windows 7 insists ESP to be FAT32.  So use it.
  * `efibootmgr` is a userspace tool to manipulate `EFI/`
* Multi-boot
  * ESP: 200MB
  * WIN MSR: 128MB
  * WIN SYS: 80GB
  * LIN SYS: 20GB
  * LIN HOME: 140GB
  * WIN DATA: 80GB
  * Windows should be installed first.  Ubuntu 11.04 wipes ESP so back up ESP.
