Kernel vmscan
=============

## LRU

- there is a global array of `typedef struct pglist_data pg_data_t`
  - one `pg_data_t` for each NUMA node
- insidie `pg_data_t`, there is a `struct lruvec`
- inside `struct lruvec`, there are 5 lists
  - `LRU_INACTIVE_ANON`
  - `LRU_ACTIVE_ANON`
  - `LRU_INACTIVE_FILE`
  - `LRU_ACTIVE_FILE`
  - `LRU_UNEVICTABLE`
- `add_page_to_lru_list` adds a page to one of the lists
  - it actually adds the folio containing the page to the list

## page, page cache, and swap

- pages are allocated with `alloc_page` or `alloc_pages`
  - they are freed with `__free_page` or `__free_pages`
- in fs, pages are allocated the same way, but they are also added to the page
  cache, `add_to_page_cache`, and to the swap lru cache, `lru_cache_add`
  - it turns out pages are refcounted: `page_ref_xxx`
    - page cache owns references to the managed pages
    - swap cache also owns references to the managed pages
  - when a driver has a page that is from fs, not from `alloc_page`, the page
    should be freed with `put_page` or `pagevec_release`
  - it also has a `_mapcount` which is the number vmas the page is mapped
    - when a pte is taken down in `zap_pte_range`, `page_remove_rmap` is called
      to reduce `_mapcount`.  The referernce to the page is transferred to tlb
      in `__tlb_remove_page`
- amazingly, it also embeds a 1-bit semaphore
  - `lock_page` indicates down
  - `unlock_page` indicates up
- when a page is allocated, it can specify the reclaim flags
  - `__GFP_DIRECT_RECLAIM` means the caller would rather wait than failing the
    allocation
  - `___GFP_KSWAPD_RECLAIM` means the allocation can fail, but please wake up
    kswapd
- kswapd thread calls `kswapd_shrink_node` to reclaim pages from various
  sources to the buddy allocator
- specifically, `shrink_page_list` is given a list of pages to reclaim
  - it locks the page with `lock_page`
  - it checks `mapping_unevictable`
  - it `try_to_unmap` the pages from all vmas
  - if the page is in a page cache, and have no other references, removing it
    from the page cache and taking away the last references in
    `__remove_mapping`
  - it does many other things; if all good, the pages are added to
    `free_pages`
  - then those pages are returned to the buddy allocator with
    `free_unref_page_list`
