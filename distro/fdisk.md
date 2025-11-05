fdisk
=====

## Partitioning

- fdisk
  - `g` to use GPT
    - by default, fdisk reserves the first and the last 1MB of the disk for
      primary and secondary GPT tables
  - partition 1: 2GB, ESP
    - `mkfs.fat -F32` to force FAT32
      - otherwise, it picks one of FAT12, FAT16, or FAT32
    - FAT32 is the only required fs by uefi
  - partition 2: rest of the disk, Linux
    - use btrfs or two partitions if want to separate root and home
  - no swap
    - use zram instead
- current disk usage on a small laptop
  - /home: 55G
  - /var: 8G (down to 3G after pacman -Sc)
  - /usr: 5G
  - rest: 1G
- <https://uapi-group.org/specifications/specs/boot_loader_specification/>
  - if ESP is large enough to hold kernels, it should be mounted to `/boot`
  - if ESP is too small to hold kernels, it should be moutned to `/efi`
    - another partition, XBOOTLDR, should be created and moutned to `/boot`
      instead

## Booting

- when the cpu is powered on,
  - on x86, the cpu typically executes the firmware stored in spi nor
    - the fw performs very early init, including dram init
      - e.g., the fw can be coreboot
    - the fw loads edk2 to dram and jumps to edk2
      - e.g., edk2 can be a coreboot payload
    - edk2 picks the bootloader
      - it consults `BootOrder` variable if defined
      - otherwise, it falls back to `\EFI\BOOT\BOOTX64.EFI` on ESP
  - on arm, the cpu typically executes the bootrom stored in mask rom
    - the bootrom loads the fw from spi nor, spi eeprom, emmc, sd, etc.
      - the bootrom is read-only and can never be updated
    - the fw performs very early init, including dram init
      - e.g., the fw can be uboot spl
    - the fw loads BL31 and BL33 and jumps to BL31/BL33
      - e.g., BL31 can be atf and BL33 can be uboot proper
      - there is optional BL33 which can be op-tee
    - uboot proper can boot to edk2, or more commonly, boot to kernel directly
- edk2 finds the bootloader on ESP
  - bootloader finds its config files on ESP
    - e.g., `systemd-boot` finds `/loader/loader.conf` on ESP
  - bootloader finds bootloader entries on ESP and, if exists, XBOOTLDR
    - type #1 boot entries are defined by `/loader/entries/*.conf`
    - type #2 boot entries are ukis under `/EFI/Linux/*.efi`
  - bootloader entries refer to kernel/initramfs files using absolute paths on
    the same partitions
    - that is, a bootloader entry and its associated kernel/initramfs must be on
      the same partition
- uboot proper can itself be the bootloader
  - it finds its config file, `/extlinux/extlinux.conf`, on ESP
  - it can also load `/EFI/BOOT/BOOTAA64.EFI` on ESP like edk2 does

## Data Migration

- partition table cloning
  - `sfdisk -d /dev/sda | sfdisk /dev/sdb`
  - it is usually better to dump to a file and manually edit first
    - to generate new uuids or when the two disks have difference sizes
    - header lines
      - keep `label: gpt`
      - remove the rest and let sfdisk derive them from the new disk
    - partition lines
      - keep `type`, `size`, `name`, and `attrs`
        - remove `size` as well for the last partition
      - remove the rest and let sfdisk derive them
- partition cloning
  - the old and new partitions should have the same size
    - if the size grows, follow the cloning by `resize2fs /dev/sdb4`
  - `cp /dev/sda4 /dev/sdb4` copies the entire partition
  - `e2image -arp /dev/sda4 /dev/sdb4` copies ext partitions
    - it only copies blocks that are used
- data cloning
  - `cp -ax --sparse=always / /mnt`
    - `-a` is recursive and preserves everything (mode, owner, timestamps,
      context, links, xattr)
    - `-x` stays on the same filesystem
  - `rsync -qaHAXSx / /mnt`
    - `-q` is quiet
    - `-a` is archive (recursive and preserves most things)
    - `-H` preserves hard links
    - `-A` preserves acls
    - `-X` preserves xattrs
    - `-S` uses sparse
    - `-x` stays on the same filesystem

## Bootable USB

- <https://wiki.syslinux.org/wiki/index.php?title=Isohybrid>
- ISO 9660 filesystem
  - the first 32KB is unused
  - that is enough space for MBR or GPT
- a bootable iso
  - an ISO 9660 image with MBR or GPT
  - it is often MBR with 2 partitions
    - first partition has type "empty" and takes up the entire iso
    - second partition has type "esp" and is small
  - bios can boot it as cdrom
    - the bootloader is isolinux
  - bios can boot it as usb
    - the bootloader is isolinux
  - uefi can find the esp partition, which cotains the bootloader
    - often grub

## SBCs

- rockchip
  - <https://opensource.rock-chips.com/wiki_Partitions>
  - partition 1 is at sector 64 (0x40)
    - bootrom always finds preloader at sector 64
    - preloader initializes dram
  - partition 2 is at sector 16384 (0x4000, 8MB)
    - preloader usually finds the bootloader at sector 16384
    - common bootloader is u-boot proper, in which case
      - the preloader includes uboot spl
      - this partition also includes atf
  - partition 3 is at sector 24576 (0x6000, 12MB)
    - u-boot includes tf-a and does not need this partition
  - partition 4 is at sector 32768 (0x8000, 16MB)
    - this is `/boot` and holds kernel/initramfs/dtb as well as
      `extlinux/extlinux.conf`
- broadcom
  - vpu bootrom always finds stage2 in eeprom
  - vpu stage2 initializes dram and finds `start4.elf` on the first vfat
    partition
  - vpu `start4.elf` functions as the bootloader and boots `kernel8.img`
- partition table
  - fdisk by default considers 2048 to be the first usable sector
    - that is, it sets "First usable LBA for partitions" field in the gpt
      header to 2048
    - use `echo -e "label:gpt\nfirst-lba: 34" | sfdisk <image>`, or
      `parted <image> mklabel gpt`, to set the field to 34
  - partition 1: sector 64 to 16383
    - for rockchip preloader (ddr training and uboot spl)
  - partition 2: sector 16384 to 32767
    - for rockchip bootloader (atf and uboot proper)
  - partition 3: sector 32768, size 1GB, esp, fat32
    - for `/boot/firmware` on broadcom
    - label `RASPIFIRM`
  - partition 4: size 2GB, xbootldr, fat32
    - for `/boot`
    - label `RASPIBOOT`
  - partition 5: rest, ext4
    - for rootfs
    - label `RASPIROOT`
  - on rockchip, u-boot will find `/boot/extlinux/extlinux.conf` on partition
    4
  - on broadcom, vpu `start4.elf` will find `/boot/firmware/kernel8.img` on
    partition 3
    - kernel/initramfs must be copied from `/boot` to `/boot/firmware`
    - alternatively, `/boot/firmware/kernel8.img` can be u-boot which will
      find `/boot/extlinux/extlinux.conf` on partition 4

## GPT

- <https://en.wikipedia.org/wiki/GUID_Partition_Table>
  - disks have physical "sectors" that are commonly 512-byte or 4096-byte
  - softwares use logical "blocks" that are tradtionally 512 bytes
  - fdisk can report both
- LBA 0: protective MBR
  - reserved for limited backward compatibility with BIOS MBR
- LBA 1: partition table header
  - off 0x00: signature
  - off 0x08: revision
  - off 0x0c: header size (usually 0x5c)
  - off 0x28: first usable LBA for partitions (usually 34)
  - off 0x30: last usable LBA for partitions (usually disk size minus 34)
  - off 0x48: starting LBA for partition entries (usually 2)
  - off 0x50: number of partition entries
  - off 0x54: size of a partition entries (usually 128 bytes)
  - off 0x5c to end-of-block: mbz
- LBA 2-33: partition entries
  - each LBA traditionally hosts 4 partition entries, up to a total of 128
    partition entries
    - an LBA tradtionally has 512 bytes
    - a partition entry tradtionally has 128 bytes
  - off 0x00: partition type guid
  - off 0x10: unique partition guid
  - off 0x20: first LBA
  - off 0x28: last LBA
  - off 0x30: attribute flags
    - bit 0: must be preserved (used for OEM partitions, etc.)
    - bit 1: uefi should ignore this partition
    - bit 2: legacy bios bootable
    - bit 3..47: reserved
    - bit 48..63: partition type specific flags
  - off 0x38: partition name (72 bytes)
- LBA last 1-32: secondary partition entries
- LBA last 0: secondary partition table header
- partition type specific flags
  - Microsoft basic data partition
    - bit 60: read-only
    - bit 61: shadow copy
    - bit 62: hidden
    - bit 63: no driver letter
  - ChromeOS kernel
    - bit 48..51: priority (15 is highest, 1 is lowest, 0 is not bootable)
    - bit 52..55: tries remaining
    - bit 56: successful boot flag

## MBR

- <https://en.wikipedia.org/wiki/Master_boot_record>
- LBA 0: MBR
  - off 0x0000: bootstrap code area
  - off 0x00da: mbz
  - off 0x00da: optional disk timestamp
  - off 0x00e0: bootstrap code area
  - off 0x01b8: optional disk signature
  - off 0x01be: partition entry 1
  - off 0x01ce: partition entry 2
  - off 0x01de: partition entry 3
  - off 0x01ee: partition entry 4
  - off 0x01fe: boot signature
- partition entries
  - off 0x0: status (bit 7 is bootable)
  - off 0x1: first sector in CHS
  - off 0x4: type
  - off 0x5: last sector in CHS
  - off 0x8: first sector in LBA
  - off 0xc: size
- partition types
  - 0x00: empty
  - 0x07: NTFS/exFAT
  - 0x0c: FAT32
  - 0x0f: extended partition
  - 0x82: linux swap
  - 0x83: linux
  - 0x85: linux extended partition
  - 0x8e: linux LVM
  - 0xa5: freebsd
  - 0xa6: openbsd
  - 0xa8: darwin
  - 0xa9: netbsd
  - 0xab: darwin boot
  - 0xaf: HFS
  - 0xee: GPT
    - on a gpt disk, the first sector is protective mbr
    - it should has a single partition of this type for the entire disk
  - 0xef: ESP
    - yeah, ESP can be on a MBR disk as well

## Windows

- <https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions>
- the device can have multiple disks
  - both GPT and MBR are supported
  - the disk containing the Windows partition must use GPT
- types of partitions
  - System partition (ESP)
  - Microsoft reserved partition (MSR)
  - OEM partitions
  - Windows partition
  - Recovery tools partition
  - Data partitions
- ESP
  - must have at least 1 ESP
  - minimum size is 100MB
  - must be FAT32
    - FAT32 requires a minimum size of 260MB on 4KB-sector disks (and a
      minimum size of 36MB on 512B-sector disks)
      - because it requires a minimum of 65527 clusters plus some for metadata
    - `mkfs.fat -F 32 -S 4096` warns when the partition size is
      `65695 * 4096` or less
    - `mkfs.fat -F 32 -S 4096 -a -f 1 -h 0 -R 2 -s 1` warns when the
      partition size is `65590 * 4096` or less
    - `mkfs.fat -F 32` warns when the partition size is `66591 * 512` or less
- MSR
  - 16MB
  - each GPT disk should have a MSR partition
- OEM partitions
  - must locate before Windows/Resovery/Data partitions
  - must set preserved bit
- Windows partition
  - minimum size is 20GB
    - for win11, 128GB
      - <https://www.microsoft.com/en-us/windows/windows-11-specifications>
      - 64GB for installation
      - double that for updates
  - must be NTFS
  - must be on a GPT disk
- Recovery tools partition
  - minimum size is 300MB
  - should follow Windows partition immediately
- Data partitions
  - not recommended
  - should follow Recovery tool partition immediately
