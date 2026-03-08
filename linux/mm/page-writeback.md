Kernel Page Writeback
=====================

## Overview

- `do_writepages` writes dirty pages back to the backing storage
  - it is by itself straightfoward
  - `wb_bandwidth_estimate_start` and `wb_update_bandwidth` measure the
    writeback bandwidth for throttling
- explicit sync calls down to `filemap_fdatawrite_range` to trigger
  `do_writepages`
- `balance_dirty_pages_ratelimited` is called on paths that dirty pages, such
  as userspace `write`
  - it uses to wb bandwidth measurement to decide if it should trigger
    background writeback
  - `wb_start_background_writeback` schedules `wb_workfn`
- `wb_workfn`
  - `wb_do_writeback` writes back
    - `wb_writeback -> writeback_sb_inodes -> __writeback_single_inode -> do_writepages`
  - `dirty_writeback_interval` is 5s, meaning it triggers writeback every 5s
    as well
