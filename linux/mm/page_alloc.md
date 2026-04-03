Kernel buddy alloc
==================

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
