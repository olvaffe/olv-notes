Linux fs buffer
===============

## Overview

- `buffer_head` is superceded by `iomap` and `folio`, but is still widely used
  - it seems to sit between `block fop -> block aop -> bh -> bio`
- a bh covers exactly one fs block
  - an fs block covers one or more disk sectors
  - a page traditionally covers one or more fs blocks
    - if a fs block is larger than a page, the fs should use iomap, not bh
- `block_read_full_folio` is a generic aop `read_folio` helper
  - `folio_create_buffers` creates a bh for the folio
  - `mark_buffer_async_read`
    - `bh->b_end_io` is `end_buffer_async_read_io`
  - `submit_bh` converts bh to bio and calls `submit_bio`
    - `bio->bi_end_io` is `end_bio_bh_io_sync`
  - when the bio completes,
    - `end_bio_bh_io_sync` calls `end_buffer_async_read_io`
      - `end_buffer_async_read` calls `folio_end_read` to wake up waiters
- `block_write_full_folio` is a generic aop `writepages` helper
  - `folio_create_buffers` creates a bh for the folio
  - if `test_clear_buffer_dirty`, `mark_buffer_async_write_endio`
    - `bh->b_end_io` is `end_buffer_async_write`
  - `folio_start_writeback` marks the folio writeback
    - that is, it transitions from dirty to writeback
  - `submit_bh_wbc` converts bh to bio and calls `submit_bio`
    - `bio->bi_end_io` is `end_bio_bh_io_sync`
  - when the bio completes,
    - `end_bio_bh_io_sync` calls `end_buffer_async_write`
      - `folio_end_writeback` wakes up waiters

## Buffer Cache and Page Cache

- when fs calls `sb_bread` to read a block (for fs metadata), `bdev_getblk`
  returns a bh associated with the bdev's page cache (aka buffer cache)
  - `__getblk_slow` is the slowest path
  - `grow_buffers`
    - `__filemap_get_folio` allocs folio in `bdev->bd_mapping`
    - `folio_alloc_buffers` allocs bh for the folio
  - `find_get_block_common` calls `__find_get_block_slow` to return the bh
- when `generic_file_read_iter` calls `filemap_read` to read a file,
  - `filemap_create_folio` is one of the paths
    - `filemap_alloc_folio` allocs folio in `filp->f_mapping`
    - `filemap_read_folio` calls `a_ops->read_folio` which asks the fs to read
      into the folio
- `/proc/meminfo`
  - `Buffers` is from `nr_blockdev_pages`
    - it sums folios in each of the bdev's page cache
  - `SwapCached` is `NR_SWAPCACHE`
    - `__swap_cache_add_folio` and `__swap_cache_del_folio` update the stat
    - that is, it counts folios in the special page cache for anon vma (aka
      swap cache)
  - `Cache` is `NR_FILE_PAGES` minus the two above
    - `NR_FILE_PAGES`
      - `__filemap_add_folio` and `__filemap_remove_folio` update the stat
      - `__swap_cache_add_folio` and `__swap_cache_del_folio` update the stat
      - `shmem_update_stats` updates the stat
      - that is, it counts folios in regular page caches (for files and
        bdevs), special page cache (for anon vma), and shmem
