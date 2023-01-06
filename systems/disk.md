Distro Disk
===========

## Partitioning

- fdisk 
  - `g` to use GPT
  - partition 1: 260M, ESP
  - partition 2: entire disk, Linux
  - by default, fdisk reserves the first and the last 1MB of the disk for
    primary and secondary GPT tables
- mkfs
  - partition 1: `mkfs.vfat -F32`
  - partition 2: `mkfs.btrfs`
- prepare btrfs
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

## Mounting

- mount partition 2 (uses `subvol=roots/current` by default)
- mount partition 1 to boot
- mount partition 2 to home with `subvol=homes/current`

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
    - FAT32 requires a minimum size of 260MB on 4KB-sector disks
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
