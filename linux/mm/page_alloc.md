# Kernel buddy alloc

## Initialization

- `memblock_free_all` hands pages over to buddy
  - `free_unused_memmap` is nop on x86/arm
  - `reset_all_zones_managed_pages` resets `zone->managed_pages`
  - `free_low_memory_core_early`
    - `memmap_init_reserved_pages` marks pages in reserved regions as
      `PG_reserved`
    - `__free_memory_core` hands pages in each non-reserved region to buddy
      - `memblock_free_pages` calls `__free_pages_core`
        - it clears `PG_reserved` and refcount
        - it increments `zone->managed_pages`
        - `__free_pages_ok`
  - `totalram_pages_add` updates stats
- `free_initmem` hands more pages over to buddy
  - these are pages for kernel `__init` section
  - `free_reserved_area` calls `free_reserved_page` to hand over

## `alloc_pages`

- `alloc_pages -> alloc_pages_noprof -> alloc_frozen_pages_noprof ->
  alloc_pages_mpol -> __alloc_frozen_pages_noprof`
  - a frozen page has refcount == 0
  - `alloc_pages_noprof` calls `set_page_refcounted` to set refcount to 1
- `prepare_alloc_pages` preps an `alloc_context`
  - `gfp_to_alloc_flags_cma` casts gfp flags to alloc flags
- `get_page_from_freelist` is the fast path
  - it loops over allowed zones and checks against watermarks
  - `rmqueue` tries to remove a free page from a zone
    - `rmqueue_pcplist` tries per-cpu list, `zone->per_cpu_pageset`, first
      - if empty, `rmqueue_bulk` tries to refill the per-cpu list
    - `rmqueue_buddy` calls `__rmqueue` which calls `__rmqueue_smallest`
      - it removes a page from `zon->free_area[current_order].freelist[migratetype]`
  - `prep_new_page` preps the page struct before returning
    - `post_alloc_hook` sanitizes the page
    - if `__GFP_COMP`, `prep_compound_page` preps for compound page
- `__alloc_pages_slowpath` is the slow path
  - if `ALLOC_KSWAPD`, `wake_all_kswapds` wakes up kswapd
  - `get_page_from_freelist` tries the fast path again with adjusted flags
  - if `__GFP_DIRECT_RECLAIM`, `__alloc_pages_direct_reclaim`
    - `__perform_reclaim` calls `try_to_free_pages` to reclaim
    - `get_page_from_freelist` again
  - if huge and `__GFP_IO`, `__alloc_pages_direct_compact`
    - `try_to_compact_pages` migrates pages to defragment and captures a freed page
      - `isolate_migratepages` isolates pages from zone head
      - `migrate_pages` migrates isolated pages from zone head to zone tail
        - `compaction_alloc` allocs from zone tail
        - `migrate_folio_done` drops migrated pages
          - `folio_put -> ... -> __free_frozen_pages`
      - `lru_add_drain_cpu_zone -> drain_local_pages > ... -> __free_one_page`
        - `compaction_capture` captures a freed page
  - `__alloc_pages_may_oom`
    - `get_page_from_freelist` tries the fast path with `ALLOC_WMARK_HIGH`
    - `out_of_memory` oom kills
    - `__alloc_pages_cpuset_fallback` tries the fast path with
      `ALLOC_NO_WATERMARKS`

## Page Free

- `__free_pages_ok`
  - `__free_pages_prepare` sanitizes `struct page` struct
  - `free_one_page -> split_large_buddy`
    - `pageblock_order` is typically 2MB, size of huge pages
    - it splits the range to be no more than `pageblock_order`, because there
      is no benefit
    - `get_pfnblock_migratetype` gets the migration type
    - it calls `__free_one_page`
- `__free_one_page`
  - `account_freepages` updates `NR_FREE_PAGES` and more
  - it merges the page with its buddy, to defragment
  - `set_buddy_order` sets the final order
  - `__add_to_free_list` adds the page to `zone->free_area` and increments
    `area->nr_free`

## GFP Flags

- GFP stands for "Get Free Pages"
- zone modifiers
  - `__GFP_DMA` uses `ZONE_DMA` (instead of `ZONE_NORMAL`) or below
    - `prepare_alloc_pages` inits `ac->highest_zoneidx` to `ZONE_DMA`
  - `__GFP_HIGHMEM` uses `ZONE_HIGHMEM` or below
  - `__GFP_DMA32` uses `ZONE_DMA32` or below
  - `__GFP_MOVABLE` uses `ZONE_MOVABLE` or below only when combined with
    `__GFP_HIGHMEM`
    - `gfp_zone` ignores `__GFP_MOVABLE` unless `__GFP_HIGHMEM` is also set
    - `ZONE_MOVABLE` also causes `gfp_migratetype` to return `MIGRATE_MOVABLE`
- page mobility and placement hints
  - `__GFP_RECLAIMABLE` uses `MIGRATE_RECLAIMABLE`
    - `gfp_migratetype` looks at `__GFP_MOVABLE` and `__GFP_RECLAIMABLE`
      - `MIGRATE_UNMOVABLE` is the default
      - `MIGRATE_MOVABLE` if `__GFP_MOVABLE`
      - `MIGRATE_RECLAIMABLE` if `__GFP_RECLAIMABLE`
  - `__GFP_WRITE` hints the page will be dirtied
    - `prepare_alloc_pages` inits `ac->spread_dirty_pages` to true
    - `get_page_from_freelist` balances page allocation between nodes, to
      prevent one node from having too many dirty pages
  - `__GFP_HARDWALL` enforces alloc from allowed nodes when `cpusets_enabled`
    - cpuset cg allows userspace to config allowed nodes via `cpuset.mems`
      - `cpuset_write_resmask -> update_nodemask -> update_nodemasks_hier ->
        cpuset_update_tasks_nodemask -> cpuset_change_task_nodemask`
    - `__cpuset_zone_allowed` checks if a node is allowed and `__GFP_HARDWALL`
      strictly enforces the allowed nodes
  - `__GFP_THISNODE` enforces alloc from local node
    - `node_zonelist` returns `ZONELIST_NOFALLBACK` instead of
      `ZONELIST_FALLBACK`
    - the lists are built by `build_all_zonelists` during init
  - `__GFP_ACCOUNT` charges memcg if active
    - `__alloc_frozen_pages_noprof` charges to memcg
  - `__GFP_NO_OBJ_EXT` is used for slab `CONFIG_MEM_ALLOC_PROFILING`
    - it breaks the `kmalloc -> alloc_tagging_slab_alloc_hook -> kmalloc` loop
- watermark modifiers
  - `__GFP_HIGH` allows tapping into emergency reserve with limit
    - `gfp_to_alloc_flags` casts `__GFP_HIGH` to `ALLOC_MIN_RESERVE`
  - `__GFP_MEMALLOC` allows tapping into emergency reserve with no limit
    - `__gfp_pfmemalloc_flags` returns `ALLOC_NO_WATERMARKS`
  - `__GFP_NOMEMALLOC` forbids tapping into emergency reserve
    - `__gfp_pfmemalloc_flags` returns 0 even when process-wide `PF_MEMALLOC`
      is set
- reclaim modifiers
  - `__GFP_IO` allows direct reclaim to trigger io
    - `gfp_compaction_allowed` can compact
    - `shrink_folio_list` can swap out anon pages or write back dirty pages
  - `__GFP_FS` allows direct reclaim to trigger fs
    - if an fs allocs with `__GFP_FS` while holding a lock, direct reclaim
      can potentially deadlock
    - `throttle_direct_reclaim` can block on kswapd forever
    - `shrink_folio_list` can write back dirty pages
  - `__GFP_DIRECT_RECLAIM` allows direct reclaim
    - `__alloc_pages_slowpath` checks the flag before reclaiming
  - `__GFP_KSWAPD_RECLAIM` allows waking up kswapd
    - `gfp_to_alloc_flags` casts `__GFP_KSWAPD_RECLAIM` to `ALLOC_KSWAPD`
    - `__alloc_pages_slowpath` calls `wake_all_kswapds`
  - `__GFP_RECLAIM` is `___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM`
  - `__GFP_RETRY_MAYFAIL` retries direct reclaims for high-order allocations
    and prefers failing over oom killer
    - `__alloc_pages_slowpath` retries for `costly_order`
    - `__alloc_pages_may_oom` fails over oom killer
  - `__GFP_NOFAIL` never fails (dangerous!)
    - `__alloc_pages_slowpath` retries forever if direct reclaim is allowed
    - `__alloc_pages_may_oom` taps into emergency reserve with no limit
  - `__GFP_NORETRY` prefers fast failing if no free page
    - `__alloc_pages_slowpath` direct reclaims lightly, never retries, never
      oom killer
- action modifiers
  - `__GFP_NOWARN` suppresses alloc fail warning
    - `__alloc_pages_slowpath` calls `warn_alloc` to warn on alloc failures
    - when the caller expects and handles occasional failures, it can set the
      flag
  - `__GFP_COMP` allocs a composite page
    - after high-order pages are allocated, `prep_new_page` calls
      `prep_compound_page` to set up composite page metadata
    - `folio_alloc` always sets `__GFP_COMP`
  - `__GFP_ZERO` zeros newly allocated pages
    - `want_init_on_alloc` returns true
  - `__GFP_ZEROTAGS` zeros memory tags if pages are zeroed
    - `post_alloc_hook` calls `tag_clear_highpages` to zero tags and pages as
      an optimization if supported
  - `__GFP_SKIP_ZERO` disables `__GFP_ZERO`
    - kasan sets the bit to defer vmalloc zeroing to `KASAN_VMALLOC_INIT` as
      an optimization
  - `__GFP_SKIP_KASAN` skips kasan
    - if a page is allocated for kernel, the bit is not set
      - `kasan_unpoison_pages` unpoisons the page
        - `kasan_unpoison` assigns memory tag
        - `page_kasan_tag_set` assigns page (pointer) tag
          - uaf via kernel linear mapping can be detected
    - if a page is allocated for userspace, the bit should be set
      - userspace, instead of kernel, will manage memory tag
      - `page_kasan_tag_reset` resets page (pointer) tag
        - uaf via kernel linear mapping cannot be detected
  - `__GFP_NOLOCKDEP` disables lockdep check for `__GFP_FS`
    - `fs_reclaim_acquire` skips the check
- convenience combos
  - `GFP_ATOMIC` is `__GFP_HIGH|__GFP_KSWAPD_RECLAIM`
  - `GFP_KERNEL` is `__GFP_RECLAIM|__GFP_IO|__GFP_FS`
  - `GFP_KERNEL_ACCOUNT` is `GFP_KERNEL|__GFP_ACCOUNT`
  - `GFP_NOWAIT` is `__GFP_KSWAPD_RECLAIM|__GFP_NOWARN`
  - `GFP_NOIO` is `__GFP_RECLAIM`
  - `GFP_NOFS` is `__GFP_RECLAIM|__GFP_IO`
  - `GFP_USER` is `__GFP_RECLAIM|__GFP_IO|__GFP_FS|__GFP_HARDWALL`
  - `GFP_DMA` is `__GFP_DMA`
  - `GFP_DMA32` is `__GFP_DMA32`
  - `GFP_HIGHUSER` is `GFP_USER|__GFP_HIGHMEM`
  - `GFP_HIGHUSER_MOVABLE` is `GFP_HIGHUSER|__GFP_MOVABLE|__GFP_SKIP_KASAN`
  - `GFP_TRANSHUGE_LIGHT` is `(GFP_HIGHUSER_MOVABLE|__GFP_COMP| __GFP_NOMEMALLOC|__GFP_NOWARN) & ~__GFP_RECLAIM`
  - `GFP_TRANSHUGE` is `GFP_TRANSHUGE_LIGHT|__GFP_DIRECT_RECLAIM`

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

## Page Refcounts

- `alloc_page` and `__free_page`
  - `alloc_page` allocs a page with refcount 1
    - `alloc_frozen_pages_noprof` allocs a page with refcount 0
    - `set_page_refcounted` sets refcount to 1
  - `__free_page` frees a page with refcount 1
    - `put_page_testzero` decrements refcount to 0
    - `__free_frozen_pages` frees a page with refcount 0
- `vm_operations_struct::fault`
  - `filemap_fault`, the generic implementation
    - `filemap_get_folio` returns a folio with incremented refcount
    - `vmf->page` is the folio
  - `do_fault`, the user
    - `__do_fault` calls the callback to get `vmf->page`
    - `finish_fault` assigns the folio to the pte entry, which is the owner
    - on errors, `folio_put` decrements the refcount
- `mm_struct::pgd`
  - the pgd table, all intermediate levels of tables, and the leaf pages form
    a tree
  - all pages are owned by the tree
  - `exit_mmap` frees all but the pgd table
    - `unmap_vmas` frees leaf pages and clears pte tables
    - `free_pgtables` frees all but the pgd table
  - `mm_free_pgd` frees the pgd table
- `filemap_add_folio` and `filemap_remove_folio`
  - `filemap_add_folio` calls `__filemap_add_folio`
    - `folio_ref_add` adds a ref
  - `filemap_remove_folio` calls `filemap_free_folio`
    - `folio_put_refs` removes a ref
- `folio_add_file_rmap_ptes` and `folio_remove_rmap_ptes`
  - they don't change refcount
- `folio_add_lru`
  - it does not change refcount
  - when `folio_put` drops the last ref, `page_cache_release` calls
    `lruvec_del_folio` automatically (but not atomically; the refcount is
    decremented first)

## Page Mapcounts

- the page also has a `_mapcount` which is the number vmas the page is mapped
  - when a pte is taken down in `zap_pte_range`, `page_remove_rmap` is called
    to reduce `_mapcount`.  The referernce to the page is transferred to tlb
    in `__tlb_remove_page`
