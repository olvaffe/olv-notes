Kernel Block Devices
====================

## loop

- `loop_init`
  - `misc_register` registers `/dev/loop-control`
    - `LOOP_CTL_ADD` adds a `loop_device`
    - `LOOP_CTL_REMOVE` removes a `loop_device`
    - `LOOP_CTL_GET_FREE` adds a `loop_device` with auto-assigned index
  - `__register_blkdev` reserves `LOOP_MAJOR`
  - `loop_add` pre-adds `CONFIG_BLK_DEV_LOOP_MIN_COUNT` loop devices
- `loop_add`
  - it allocs a `loop_device`
  - `blk_mq_alloc_disk` allocs `lo->lo_disk`
    - `disk_name` is set to `loop%d`
    - `fops` is set to `lo_fops`
  - `add_disk` adds `lo->lo_disk`
- `lo_ioctl` performs an ioctl on `/dev/loop%d`
  - `LOOP_CONFIGURE` configures the dev
