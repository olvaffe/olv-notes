Kernel buddy alloc
==================

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
