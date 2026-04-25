# Linux fs-writeback

## Overview

- `wb_workfn` may be scheduled by
  - `mm -> balance_dirty_pages_ratelimited -> wb_start_background_writeback`
  - `mm -> vmscan -> wakeup_flusher_threads -> wb_start_writeback`
  - `vfs -> ... -> wb_queue_work`
  - `__mark_inode_dirty -> wb_wakeup_delayed`
  - `wb_workfn -> wb_wakeup_delayed`
  - etc.
- `wb_workfn` always calls `wb_do_writeback`
  - `wb_writeback` executes a work
    - if there are real queued works, they are executed first
    - if `WB_start_all`, `wb_check_start_all` executes a faked work
    - if it has been `dirty_writeback_interval` (5s),
      `wb_check_old_data_flush` executes a faked work
    - if dirty pages are more than a certain threshold,
      `wb_check_background_flush` executes a faked work
  - `queue_io` gathers dirty inodes to `wb->b_io`
  - if the work is scoped to an sb, `writeback_sb_inodes` writes back dirty
    inodes belonging to `work->sb`
    - `__writeback_single_inode` calls `do_writepages`
  - otherwise, `__writeback_inodes_wb` writes back all dirty inodes
