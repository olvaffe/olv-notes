Kernel rmap
===========

## Overview

- it provides reverse mapping
  - given a page, find all vmas (from all processes) that the page is mapped
  - this is needed for page reclaim, page migration, etc.
- `rmap_walk` is the main function
  - if a folio belongs to the page cache of a file, `folio->mapping` points to
    the page cache
    - `folio->mapping->i_mmap` tracks all vmas
  - if a folio belongs to an anonymous mapping (no backing file),
    `folio->mapping` points to a `anon_vma`
    - `folio->mapping->rb_root` tracks all vmas
- `try_to_unmap` calls `rmap_walk` to unmap a page from all vmas for reclaim
