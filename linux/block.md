# Linux block layer

## Initialization

- `subsys_initcall(genhd_device_init)`
- `class_register(&block_class)` registers `/sys/class/block`
  - `blk_mq_alloc_disk` allocs a `gendisk` of this class
  - `add_partition` adds a `block_device` of this class
- `register_blkdev(BLOCK_EXT_MAJOR, "blkext")` reserves major 259
- `kobject_create_and_add("block", NULL)` creates `/sys/block`
  - `device_add_disk` will create symlink under this directory

## Controller Driver

- when a controller is probed,
  - `blk_mq_alloc_tag_set` allocs a `blk_mq_tag_set`
- when a storage on the controller is discovered,
  - `blk_mq_alloc_disk` allocs a `gendisk` from the tag set
    - `blk_mq_alloc_queue` allocs a `request_queue`
    - `__alloc_disk_node` allocs a `gendisk` associated with the request queue
      - `bdi_alloc` allocs `disk->bdi`, a `backing_dev_info` for mm
      - `bdev_alloc` allocs `disk->part0`, a `block_device` representing the
        whole gendisk
        - it is an inode and has a pagecache
        - it is also a `device` in the device mode
      - `disk->part0` is added to `disk->part_tbl`
      - `device_initialize` inits `disk->part0->bd_device`
  - `device_add_disk` adds the gendisk to the block layer
    - `__add_disk`
      - `device_add` adds `disk->part0->bd_device`
      - `bdi_register` adds `disk->bdi` to mm
    - `add_disk_final`
      - `bdev_add` adds `disk->part0`
      - `disk_scan_partitions` calls `bdev_open` to trigger synchronous
        partition scan
- when `submit_bio` submits a bio,
  - `blk_mq_submit_bio` submits the bio indirectly to the driver
    - `blk_mq_bio_to_request` converts bio to `request`
    - `blk_mq_insert_request` inserts the request to the queue
    - `blk_mq_run_hw_queue` runs the queue
      - `blk_mq_sched_dispatch_requests -> blk_mq_dispatch_rq_list` dispatches
        requests
        - `mq_ops->queue_rq` queues a request to controller
        - `mq_ops->commit_rqs` commits requests to controller
- partition scanning
  - `device_add_disk -> add_disk_final -> disk_scan_partitions` triggers scan
  - `blkdev_get_whole -> bdev_disk_changed -> blk_add_partitions` adds parts
    - `check_partition` tries
      - `efi_partition` for gpt
      - `msdos_partition` for dos
    - `blk_add_partition` calls `add_partition` to add a part
      - `bdev_alloc` allocs a `block_device` for the part
        - it is both a `device` and an inode
      - `device_initialize` and `device_add` add it to the device model
      - the bdev is added to `disk->part_tbl`
  - read op submission
    - fop: `blkdev_read_iter -> filemap_read -> filemap_get_pages -> filemap_update_page -> filemap_read_folio`
    - aop: `blkdev_read_folio -> block_read_full_folio -> submit_bh`
    - bio: `submit_bio -> blk_mq_submit_bio -> blk_mq_run_hw_queue`
    - req: `blk_mq_sched_dispatch_requests -> blk_mq_dispatch_rq_list`
    - `filemap_read_folio` calls `folio_wait_locked_killable` to wait
  - read op completion
    - req: `irq -> blk_mq_end_request -> blk_update_request`
    - bio: `bio_endio -> end_bio_bh_io_sync`
    - aop: `end_buffer_async_read_io -> end_buffer_async_read -> folio_end_read`
    - `folio_end_read` calls `folio_wake_bit` to wake
