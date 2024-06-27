Distro Disk
===========

## Partitioning

- fdisk 
  - `g` to use GPT
    - by default, fdisk reserves the first and the last 1MB of the disk for
      primary and secondary GPT tables
  - partition 1: 1GB, ESP
  - partition 2: rest of the disk, Linux
    - use two partitions if want to separate root and home
  - no swap
    - use zram instead
- mkfs
  - partition 1: `mkfs.vfat -F32`
  - partition 2: `mkfs.btrfs`
    - mount partition 2
    - `mkdir roots homes`
    - `btrfs subvolume create roots/current`
    - `mkdir roots/current/{boot,home}`
    - `btrfs subvolume create homes/current`
    - `btrfs subvolume set-default roots/current`
    - umount
- current disk usage on a small laptop
  - /home: 55G
  - /var: 8G (down to 3G after pacman -Sc)
  - /usr: 5G
  - rest: 1G

## Booting

- EFI finds the bootloader on ESP
- bootloader finds its config files on ESP
  - e.g., `systemd-boot` finds `/loader/loader.conf` on ESP
- bootloader finds bootloader entries (`/loader/entries/*.conf`) on ESP and,
  if exists, XBOOTLDR
- bootloader entries refer to kernel/initramfs files using absolute paths on
  the same partitions
  - that is, a bootloader entry and its associated kernel/initramfs must be on
    the same partition
- <https://uapi-group.org/specifications/specs/boot_loader_specification/>
  - if ESP is large enough to hold kernels, it should be mounted to `/boot`
  - if ESP is too small to hold kernels, it should be moutned to `/efi`
    - another partition, XBOOTLDR, should be created and moutned to `/boot`
      instead

## Mounting

- mount partition 2 (uses `subvol=roots/current` by default)
- mount partition 1 to `/boot`
- mount partition 2 to `/home` with `subvol=homes/current`

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
    - `mkfs.vfat -F 32 -S 4096` warns when the partition size is
      `65695 * 4096` or less
    - `mkfs.vfat -F 32 -S 4096 -a -f 1 -h 0 -R 2 -s 1` warns when the
      partition size is `65590 * 4096` or less
    - `mkfs.vfat -F 32` warns when the partition size is `66591 * 512` or less
- MSR
  - 16MB
  - each GPT disk should have a MSR partition
- OEM partitions
  - must locate before Windows/Resovery/Data partitions
  - must set preserved bit
- Windows partition
  - minimum size is 20GB
  - must be NTFS
  - must be on a GPT disk
- Recovery tools partition
  - minimum size is 300MB
  - should follow Windows partition immediately
- Data partitions
  - not recommended
  - should follow Recovery tool partition immediately

## SBCs

- rockchip
  - <https://opensource.rock-chips.com/wiki_Partitions>
  - partition 1 is at sector 64 (0x40)
    - bootrom always finds preloader at sector 64
    - preloader initializes dram
  - partition 2 is at sector 16384 (0x4000, 8MB)
    - preloader usually finds the bootloader at sector 16384
    - common bootloader is u-boot
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
  - fdisk considers 2048 to be the first usable sector and we cannot have
    partitions starting before sector 2048
  - reserved: sector 64 to 16383
    - for rockchip preloader
  - reserved: sector 16384 to 32767
    - for rockchip u-boot
  - partition 1: sector 32768, size 260MB, esp, fat32
    - for `/boot/firmware` on broadcom
    - label `RASPIFIRM`
  - partition 2: size 260MB, xbootldr, fat32
    - for `/boot`
    - label `RASPIBOOT`
  - partition 3: rest, ext4
    - for rootfs
    - label `RASPIROOT`
  - on rockchip, u-boot will find `/boot/extlinux/extlinux.conf` on partition
    2
  - on broadcom, vpu `start4.elf` will find `/boot/firmware/kernel8.img` on
    partition 1
    - kernel/initramfs must be copied from `/boot` to `/boot/firmware`
    - alternatively, `/boot/firmware/kernel8.img` can be u-boot which will
      find `/boot/extlinux/extlinux.conf` on partition 2

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

## dm-crypt / LUKS

- wipe partition
  - `cryptsetup open --type plain -d /dev/urandom <part> to_be_wiped`
  - `dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress`
  - `cryptsetup close to_be_wiped`
- format the partition to LUKS
  - `cryptsetup -y -v luksFormat <part>`
- open/close the partition
  - `cryptsetup open <part> cryptroot`
  - `mkfs.btrfs /dev/mapper/cryptroot`
  - `cryptsetup close cryptroot`
- initramfs
  - distro specific
  - on arch,
    - make sure keyboard and encrypt hooks are enabled
    - `cryptdevice=UUID=<part-uuid>:cryptroot root=/dev/mapper/cryptroot`
- LVM on LUKS
  - `pvcreate /dev/mapper/cryptroot`
  - `vgcreate foo /dev/mapper/cryptroot`
  - `lvcreate -L 100%ORIGIN foo -n bar`
  - `mkfs.btrfs /dev/foo/bar`
  - for arch, make sure lvm2 hook is enabled in initramfs

## LVM

- `lvm pvscan` lists physical volumes
- `lvm vgscan` lists volume groups
- `lvm lvscan` lists logical volumes

## btrfs

- butter fs (or better fs)
- a filesystem can span multiple devices and have subvolumes
- In btrfs, a filesystem, a device, a subvolume, or an inode are all btrfs objects.
- `btrfs property` can set properties of objects.
  - `btrfs property list -t f /`
  - `btrfs property list -t d /`
  - `btrfs property list -t s /`
  - `btrfs property list -t i /`
- `btrfs filesystem` manipulates the whole filesystem
- multiple devices
  - A filesystem can be created on multiple devices.
    - `mkfs.btrfs`'s `-d` and `-m` specify how data and metadata are arranged
      on multiple devices
  - `btrfs device` can add/remove devices dynamically
- subvolumes
  - A subvolume looks like a directory in the filesystem, but can be mounted
    independently and, more importantly, can be snapshotted
  - a possible filesystem layout is
    - `/' is not mounted
    - `/roots/current` is a subvolume mounted to `/`
    - `/homes/current is a subvolume mounted to `/home`
    - to snapshot root,
      - `mount -osubvol=/ <dev> /mnt`
      - `btrfs subvolume snapshot -r / /mnt/roots/<timestamp>`
    - to roll back root,
      - `mount -osubvol=/ <dev> /mnt`
      - `btrfs subvolume delete /mnt/roots/current`
      - `btrfs subvolume snapshot /mnt/roots/<timestamp> /mnt/roots/current`

## Sync

- syscalls
  - `fsync(fd)` makes sure all changes have reached the permanent storage
    device
    - it transfers all modified buffer cache pages (data and metadata) to the
      device, waits for the transfers to finish, and flushes the device cache
      - watch out for buggy fs that does not flush device cache
      - watch out for buggy disk firmware...
    - if this is a new file in a directory, fsync on the directory is also
      needed
  - `fdatasync(fd)` is similar to `fsync(fd)` except that it does not sync
    non-essential metadata such as `st_atime` or `st_mtime`
  - `syncfs(fd)` transfers all modifications to the filesystem the file
    resides in, waits for transfers to finish, and flushes the device cache.
    - linux specific
  - `sync()` "schedules" all modifications to all filesystems to be
    transferred
    - that is what POSIX says
    - linux waits and flushes
- `sync` command
  - `sync <file>` does `fsync()`
  - `sync --data <file>` does `fdatasync()`
  - `sync --file-system <file>` does `syncfs()`
  - `sync` or `sync --file-system` does `sync()`
- conclusion
  - on Linux, `sync` is enough
  - watch out for buggy fs
  - watch out for buggy disk controller

## Tools

- lsblk
- blkid
- findmnt

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
  - the two partitions should have the same size
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

## Filesystems

- bootloader
  - `CONFIG_FAT_FS` for uefi esp
- root
  - `CONFIG_EXT4_FS` the most common
  - `CONFIG_BTRFS_FS` more feature-rich
  - `CONFIG_BCACHEFS_FS` new and unstable
  - `CONFIG_XFS_FS` default on RHEL
  - `CONFIG_F2FS_FS` designed for ssd, emmc, and sd
  - `CONFIG_ZFS` out-of-tree
- read-only and overlay
  - `CONFIG_SQUASHFS` the most common
  - `CONFIG_EROFS_FS` faster, default on android
  - `CONFIG_CRAMFS` obsoleted, precursor to squashfs
  - `CONFIG_ROMFS_FS` too limited
  - `CONFIG_ISO9660_FS` cd
  - `CONFIG_OVERLAY_FS` write overlay
- mtd
  - `CONFIG_JFFS2_FS` the most common
  - `CONFIG_UBIFS_FS` newer and better
- network
  - `CONFIG_NFS_FS` the most common
  - `CONFIG_SMBFS` cross-platform file sharing
  - `CONFIG_9P_FS` vm file sharing
- windows
  - `CONFIG_EXFAT_FS` modern and cross-platform
  - `CONFIG_NTFS3_FS` read-write
  - `CONFIG_NTFS_FS` older, read-only, partial write
- osx
  - `CONFIG_APFS` out-of-tree
  - `CONFIG_HFSPLUS_FS` predate osx
- bsd
  - `CONFIG_UFS_FS` the most common
- encryption
  - `CONFIG_ECRYPT_FS` stacked, don't use
    - use fscrypt-enabled fs or dm-crypt instead
