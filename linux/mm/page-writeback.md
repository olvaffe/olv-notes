Kernel Page Writeback
=====================

## Overview

- `balance_dirty_pages_ratelimited` is called on all paths that dirty pages,
  such as userspace `write`
  - `current->nr_dirties` tracks how many pages the thread has dirtied
  - if it is above a certain threshold, `balance_dirty_pages` is called
    - the threadhold is based on `current->nr_dirtied_pause`,
      `wb->dirty_exceeded`, and per cpu `bdp_ratelimits`
- `balance_dirty_pages` throttles until number of dirty pages is balanced
  - it has a loop to wait until balanced
  - `balance_domain_limits` derives limits from global stats
    - `balance_domain_limits`
      - `dtc->avail` is from `global_dirtyable_memory`
      - `dtc->dirty` is from `global_node_page_state(NR_FILE_DIRTY)` plus
        `global_node_page_state(NR_WRITEBACK)`
    - `domain_dirty_limits`
      - `dtc->thres` is derived from `vm_dirty_bytes`
      - `dtc->bg_thres` is derived from `dirty_background_bytes`
    - `domain_dirty_freerun`
      - `dtc->freerun` is derived from `dtc->thres` and `dtc->bg_thres`
  - if global dirty pages is above `gdtc->bg_thres`,
    `wb_start_background_writeback` schedules `wb_workfn` for background
    writeback
  - if `gdtc->freerun`, no throttoling needed
  - `balance_wb_limits` derives limits from bdev wb stats
  - if it has been `BANDWIDTH_INTERVAL` (200ms), `__wb_update_bandwidth`
    updates bdev wb stats
  - `io_schedule_timeout` sleeps for `pause` jiffies
- when background writeback runs, `wb_workfn` calls `do_writepages` to write
  dirty pages back to the backing storage
  - `mapping->a_ops->writepages` writes dirty pages to the backing storage
    - it translates offsets in the inode to blocks on bdev, and submits bios
  - `wb_update_bandwidth` updates bdev wb stats to help throttle decision
- explicit sync syscalls also call down to `filemap_fdatawrite_range` to call
  `do_writepages`
