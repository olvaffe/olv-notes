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
