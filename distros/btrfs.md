btrfs
=====

## Overview

- butter fs (or better fs)
- a filesystem can span multiple devices and have subvolumes
- In btrfs, a filesystem, a device, a subvolume, or an inode are all btrfs objects.
- `btrfs property` can set properties of objects.
  - `btrfs property list -t f /`
  - `btrfs property list -t d /`
  - `btrfs property list -t s /`
  - `btrfs property list -t i /`

## Filesystem

- `btrfs filesystem` manipulates the whole filesystem

## Devices

- A filesystem can be created on multiple devices.
  - `mkfs.btrfs`'s `-d` and `-m` specify how data and metadata are arranged on
    multiple devices
- `btrfs device` can add/remove devices dynamically

## Subvolumes

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
