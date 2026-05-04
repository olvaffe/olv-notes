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

## page alloc

- pages are allocated with `alloc_page` or `alloc_pages`
  - they are freed with `__free_page` or `__free_pages`
- when a page is allocated, it can specify the reclaim flags
  - `__GFP_DIRECT_RECLAIM` means the caller would rather wait than failing the
    allocation
  - `___GFP_KSWAPD_RECLAIM` means the allocation can fail, but please wake up
    kswapd
- amazingly, it also embeds a 1-bit semaphore
  - `lock_page` indicates down
  - `unlock_page` indicates up
- it turns out pages are refcounted: `page_ref_xxx`; for example,
  - page cache owns references to the managed pages
  - lru also owns references to the managed pages
- the page also has a `_mapcount` which is the number vmas the page is mapped
  - when a pte is taken down in `zap_pte_range`, `page_remove_rmap` is called
    to reduce `_mapcount`.  The referernce to the page is transferred to tlb
    in `__tlb_remove_page`

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
