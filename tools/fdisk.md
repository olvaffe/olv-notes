fdisk
=====

## Partitioning

- fdisk
  - `g` to use GPT
    - by default, fdisk reserves the first and the last 1MB of the disk for
      primary and secondary GPT tables
    - iow, fdisk defaults `first usable LBA for partitions` in gpt header to
      2048
    - use `echo -e "label:gpt\nfirst-lba: 34" | sfdisk <image>`, or
      `parted <image> mklabel gpt`, to set the field to 34 if necessary
  - partition 1: 2GB, ESP
    - `mkfs.fat -F32` to force FAT32
      - otherwise, it picks one of FAT12, FAT16, or FAT32
    - FAT32 is the only required fs by uefi
  - partition 2: rest of the disk, Linux
    - use btrfs or two partitions if want to separate root and home
  - no swap
    - use zram instead
- <https://uapi-group.org/specifications/specs/boot_loader_specification/>
  - if ESP is large enough to hold kernels, it should be mounted to `/boot`
  - if ESP is too small to hold kernels, it should be moutned to `/efi`
    - another partition, XBOOTLDR, should be created and moutned to `/boot`
      instead

## Storage Planning

- static vs variable
  - most dirs are static except for system update
  - some dirs are pseudo mountpoints
  - these dirs can be considered static
    - `/etc` if we consider config update as system update
    - `/root` if we use it only for system update
  - these dirs are variable
    - `/home` contains user home dirs
    - `/srv` contains served data
    - `/var` contains app data
- `/boot`
  - 2G, or at least 0.5G if capacity is limited
- `/`
  - on workstations, 16G suffices
  - on servers, 4G suffices
  - speed matters
- `/home`
  - on workstations with real users, both capacity and speed matter
  - on servers, it can be considered static if logged in only for system update
    - and if rootless podman uses `/var/lib/pod-foo`
- `/srv`
  - capacity matters if serving tons of data
  - speed wise, data are served with minimal processing so we want io bw to
    exceed net bw
- `/var`
  - logs, caches, spools can grow reasonably
  - on servers, some services can have huge app data
    - databases
    - containers
    - vms
- current disk usage
  - small laptop
    - /home: 55G
    - /var: 8G (down to 3G after pacman -Sc)
    - /usr: 5G
    - rest: 1G
  - router
    - /boot: 0.3G
    - /var: 0.3G
    - rest: 0.6G
  - server
    - /boot: 0.3G
    - /var: 2G
    - /home: 9G
    - /srv: 430G
    - rest: 2.2G

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
    - the src partition should be unmounted or ro
  - `e2image -arp /dev/sda4 /dev/sdb4` copies ext partitions
    - it only copies blocks that are used
    - the src partition should be unmounted or ro
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
