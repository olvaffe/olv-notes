Kernel btrfs
============

## Concepts

- linear logical space
  - the physical space spans over one or more physical devices
  - the logical space is linear and consists of chunks
  - each chunk is typically 1GB
  - each chunk has a type of data, metadata, or system
  - chunks are allocated on demand from the physical space
- block groups
  - when the filesystem needs to store some data, the allocator picks an
    existing block group or allocate a new block group
  - the relation between block groups and chunks are determined by the block
    group profiles
    - for `single`, each block group consists of 1 chunk, where the chunk is
      on any physical device
    - for `dup`, each block group consists of 2 chunks, where the 2 chunks are
      on the same physical device
    - for `raid1`, each block group consists of 2 chunks, where the 2 chunks
      are on different physical devices
  - the data is stored in each of the chunk for redundancy
  - the type of the block group, the data, and the chunk must match
    - it is one of data, metadata, or system
- balance runs specified block groups through the allocator again

## Use

- `mkfs.btrfs`
  - `btrfs subvolume create @`
    - `btrfs subvolume set-default @`
    - `mkdir -p @/{boot,home,srv,var/log}`
  - `btrfs subvolume create @home`
  - `btrfs subvolume create @srv`
  - `btrfs subvolume create @var-log`
- mount the fs to `/`, which uses `subvol=@` by default
  - mount esp to `/boot`
  - mount `subvol=@home` to `/home`
  - mount `subvol=@srv` to `/srv`
  - mount `subvol=@var-log` to `/var/log`

## btrfs userspace

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
    - `/` is not mounted
    - `/roots/current` is a subvolume mounted to `/`
    - `/homes/current` is a subvolume mounted to `/home`
    - to snapshot root,
      - `mount -osubvol=/ <dev> /mnt`
      - `btrfs subvolume snapshot -r / /mnt/roots/<timestamp>`
    - to roll back root,
      - `mount -osubvol=/ <dev> /mnt`
      - `btrfs subvolume delete /mnt/roots/current`
      - `btrfs subvolume snapshot /mnt/roots/<timestamp> /mnt/roots/current`
