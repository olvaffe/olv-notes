Distro Disk
===========

## Partitioning

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

## Mounting

- mount partition 2 (uses `subvol=roots/current` by default)
- mount partition 1 to `/boot`
- mount partition 2 to `/home` with `subvol=homes/current`

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

## Data Migration

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
