Linux MM
========

## Page Flags

- `PG_locked` means the page is locked
  - it functions as a per-page mutex
- `PG_writeback` means the page is being written back
  - it is set between `folio_start_writeback` and `folio_end_writeback`
- `PG_referenced` means the page was recently used
  - it is set on read, write, mmap, etc.
  - see `PG_active`
- `PG_uptodate` means the page holds the truth
  - it is set when the contents are synchronized with or newer than the
    backing
- `PG_dirty` means the page needs writeback
  - it is set when the contents are newer than the backing
- `PG_lru` means the page is on one of the lru lists
- `PG_head` means the page is the first of a compound page (hugepage)
- `PG_waiters` means `folio_waitqueue` of the page is non-empty
- `PG_active` means the page is on the active lru list
  - when reclaim sees
    - `+PG_active +PG_reference`: clear `PG_reference`
    - `+PG_active -PG_reference`: clear `PG_active` and move to inactive list
    - `-PG_active +PG_reference`: set `PG_active`, clear `PG_reference`, and
      move to active list
    - `-PG_active -PG_reference`: reclaim the page
- `PG_workingset` means the page is in the "working set"
  - e.g., it is set if it is re-faulted shortly after reclaim
- `PG_reserved` means the page is special
  - it tells mm to stay away from managing it: no reclaim, etc.
  - when an api expects a page, and to pass mmio addr to the api, kernel can
    fake a `PK_reserved` page?
- `PG_private` means `folio->private` has private fs data
  - when set, reclaim must ask fs to clear the bit before reclaim
- `PG_reclaim` means the page is being reclaimed
  - it is set when the reclaim requires writeback and takes time
  - it prevents another concurrnet reclaim
- `PG_swapbacked` means the page has no backing
  - it is set on allocation and never changes
- `PG_unevictable` means the page is on unevictable lru list
- `PG_dropbehind` means the page was used for streaming io and hit rate is low
  - e.g., `posix_fadvise(POSIX_FADV_NOREUSE)`
- `PG_mlocked` means the page is `mlock`ed
  - it causes `PG_unevictable` after the page is moved to the unevictable lru
    list
- `PG_readahead` means the page was paged in by readahead
  - if a page with the bit is accessed, it encourages readahead to read more
- `PG_swapcache` means the page is temporarily managed by swapcache
  - a private anonymous mapping has no backing and thus no page cache
    - that is, a process heap, stack, etc. has no `inode` backing and thus no
      `address_space`
  - to be able to page anonymous mapping out to swap device, its pages
    temporarily use swapcache as the page cache, which enable them to be paged
    out
  - as such, it is set when the page is managed by swapcache
