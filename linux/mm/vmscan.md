Kernel vmscan
=============

## Reclaims

- `node_reclaim` is a fast path
  - `may_writepage` is 0 by default
  - `may_unmap` is 0 by default
  - `may_swap` is 1
  - `priority` is 4 (`NODE_RECLAIM_PRIORITY`)
  - `shrink_node` is called to reclaim a node
    - `shrink_lruvec` reclaims from the lru
    - `shrink_slab` reclaims from the slab
- `try_to_free_pages` is a slow path
  - `may_writepage` is 1 by default
  - `may_unmap` is 1
  - `may_swap` is 1
  - `priority` is 12 (`DEF_PRIORITY`)
  - `shrink_zones` is called to reclaim zones
    - it just calls `shrink_node` like in the fast path, with a more
      aggressive `scan_control`
- `shrink_lruvec` reclaims pages in lru
  - `shrink_active_list` moves some folios from the specified active list to
    the corresponding inactive list
  - `shrink_inactive_list` reclaims folios on the specified inactive list
    - it calls `isolate_lru_folios` to move some folios on the specified
      inactive list (`LRU_INACTIVE_ANON` or `LRU_INACTIVE_FILE`) to a
      temporary list and calls `shrink_folio_list` on the temp list to reclaim
    - `move_folios_to_lru` moves those that are not reclaimed back to lru
- `shrink_folio_list` reclaims the specified folios
  - it locks the folio with `folio_trylock`
  - it checks `folio_evictable`
  - if `folio_test_anon` and `folio_test_swapbacked`, `add_to_swap` adds the
    folio to the swapcache
  - if `folio_mapped`, `try_to_unmap` unmaps the folio from all vmas
  - if `folio_test_dirty`, `pageout` writes the dirty page back using
    `writepage`
  - if the folio is in a page cache, and have no other references, removing it
    from the page cache and taking away the last references in
    `__remove_mapping`
  - if all good, the folios are added to `free_folios` and are returned to the
    buddy allocator in `free_unref_page_list`

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

## kswapd

- kswapd thread calls `kswapd_shrink_node` to reclaim pages from various
  sources to the buddy allocator
