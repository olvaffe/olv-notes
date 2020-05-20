Block Subsystem
===============

## Filesystem

- `mknod` creates an inode with `def_blk_fops` as the fops
- whenever the inode is opened, `blkdev_open` looks up the corresponding
  `block_device` from another inode in the internal `bdev` fs.
  - `__blkdev_get` associates a `gendisk` with the `block_device`
- the driver should have
  - `register_blkdev` adds a `blk_major_name` to `major_names` table
    - for tracking
  - `blk_register_region` adds a probe function to `bdev_map` table
    - the probe function calls `alloc_disk` to allocate a `gendisk`

## Loop

- TBD

## Buses

- Serial ATA
  - 3.0: 6 Gbit/s
  - ATA8-ACS command set
  - host controller usually provides a generic AHCI interface
- Serial Attached SCSI
  - SAS-4: 22.5 Gbit/s
  - SCSI command
  - host controller usually needs its own driver
- PCI-e
  - 3.0 x4: 4GB/s
  - NVMe command set
  - host controller usually provides a generic NVMe interface
- Over fabrics
  - iSCSI: SCSI command over TCP/IP
  - the host is a client.  It is called an initiator.
  - the remote has a server.  Each device on the remote is called a target.

## Block Layer

- I/O requests reach the block layer
- some drivers execute the block requests directly
  - NVM-e
  - virtio-blk (in the guest)
- Some drivers need the requested be translated to another command set first
