Linux block layer
=================

## Initialization

- `subsys_initcall(genhd_device_init)`
- `class_register(&block_class)` registers `/sys/class/block`
  - `blk_mq_alloc_disk` allocs a `gendisk` of this class
  - `add_partition` adds a `block_device` of this class
- `register_blkdev(BLOCK_EXT_MAJOR, "blkext")` reserves major 259
- `kobject_create_and_add("block", NULL)` creates `/sys/block`
  - `device_add_disk` will create symlink under this directory

## Block Layer

- I/O requests (`struct request`) reach the block layer
- some drivers execute the block requests directly
  - NVM-e
  - virtio-blk (in the guest)
- Some drivers need the requested be translated to another command set first
  - `sd_init_command` of sd

## Filesystem

- `mknod` creates an inode with `def_blk_fops` as the fops
- whenever the inode is opened, `blkdev_open` looks up the corresponding
  `block_device` from another inode in the internal `bdev` fs.
  - `__blkdev_get` finds the `gendisk` associated with the `block_device`,
    optionally allocating the disk on-demand
- the driver (e.g., sd) should do
  - `register_blkdev` adds a `blk_major_name` to `major_names` table
    - for tracking
  - `blk_register_region` adds a probe function to `bdev_map` table
    - the probe function can call `alloc_disk` to allocate a `gendisk`
      on-demand
- `blkdev_write_iter` of `def_blk_fops` calls `__generic_file_write_iter`
  - the page cache (`address_space`) sits in-between the filesystem and the
    block layer
  - `blkdev_writepage` of `def_blk_aops` writes a dirty page to the block
    device
    - it is hard to track down what's going on
    - at the bottom, it is `bio_alloc`, `submit_bio`, and
      `blk_mq_make_request`

## gendisk

- `alloc_disk`
- `disk->queue` `blk_mq_init_queue`
- `disk->disk_name`
- `disk->fops`
- `device_add_disk`
