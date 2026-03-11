Kernel vmscan
=============

## Overview

- direct reclaim
  - when `__alloc_pages` fails to alloc pages, and `___GFP_DIRECT_RECLAIM` is
    set, direct reclaim is triggered
  - `get_page_from_freelist` may call `node_reclaim` for light reclaim
    - it calls `shrink_node` without `may_writepage` nor `may_unmap`
  - `__alloc_pages_slowpath` may call `try_to_free_pages` for heavy reclaim
    - `do_try_to_free_pages` calls `shrink_zones` in a loop with decreasing
      `sc->priority`
    - `shrink_zones` calls `shrink_node` on each node
- background reclaim
  - when `__alloc_pages` falls to `__alloc_pages_slowpath` and
    `__GFP_KSWAPD_RECLAIM` is set, `wake_all_kswapds` calls `wakeup_kswapd` to
    potentially wake up `kswapd`
  - when memory too low, `pgdat_balanced` returns false and `kswapd` is woken
    - `balance_pgdat` calls `kswapd_shrink_node` with decreasing `sc->priority`
    - `kswapd_shrink_node` calls `shrink_node`
- `shrink_node`
  - if mglru, `lru_gen_shrink_node` takes a different path
  - otherwise, `shrink_node_memcgs` shrinks
  - `shrink_lruvec` shrink lru lists
  - `shrink_slab` calls into shrinkers

## `shrink_lruvec`

- `shrink_lruvec` reclaims pages in lru
  - `shrink_active_list` moves some folios from the specified active list to
    the corresponding inactive list
  - `shrink_inactive_list` reclaims folios on the specified inactive list
    - `isolate_lru_folios` moves some folios on the specified inactive list
      (`LRU_INACTIVE_ANON` or `LRU_INACTIVE_FILE`) to a temporary list
    - `shrink_folio_list` reclaims folios on the temp list
    - `move_folios_to_lru` moves those that are not reclaimed back to lru
- `shrink_folio_list` unmaps and free folios
  - it locks the folio with `folio_trylock`
  - it checks `folio_evictable`
  - `folio_check_references`
    - `folio_referenced` walks all pte entries pointing to the folio
      - it counts and clears reference (recently used) bit in the hw pte
        entries
    - `folio_test_clear_referenced` tests and clears sw `PG_referenced`
      (recently used) bit
    - if hw referenced, the folio is not reclaimed
      - `FOLIOREF_ACTIVATE` promotes it to active list
      - `FOLIOREF_KEEP` keeps it in inactive list
  - if the page is anonymous (no backing), `folio_alloc_swap` allocs space in
    swap
    - the page will be temporarily backed by swap cache
    - `folio_test_swapcache` will return true
    - `folio_mapping` will return `swap_address_space`
  - if `folio_mapped`, `try_to_unmap` unmaps the folio from all vmas
  - if `folio_test_dirty`,
    - if file-backed, `folio_set_reclaim` sets `PG_reclaim` and goto
      `activate_locked`
      - `folio_set_active` sets `PG_active`
      - the folio is not reclaimed but will be re-added to the active list
        - `wb_workfn` will writeback the dirty page at a later point
    - else, `pageout` writes the dirty folio back
      - if shmem-backed, `shmem_writeout`
      - else (anonymous), `swap_writeout` writes back to swap
        - it is temporarily backed by swap cache
  - `__remove_mapping` removes the folio from the page cache
    - `refcount` is the expected refcount of the folio
      - it is 2 for a regular page
      - 1 from page cache: `__filemap_add_folio` calls `folio_ref_add`
      - 1 from the reclaimer: `isolate_lru_folios` calls `folio_try_get`
    - `folio_ref_freeze` cmpxchgs the folio refcount to 0
    - if anonymous, `__swap_cache_del_folio` removes from swap cache
      - note that we have freeze the folio refcount to 0
      - unlike `swap_cache_del_folio`, `__swap_cache_del_folio` does not
        decrement refcount
    - else, `__filemap_remove_folio` removes from page cache
      - unlike `filemap_remove_folio`, `__filemap_remove_folio` does not
        decrement refcount
  - if all good, the folio is added to `free_folios`
  - `free_unref_folios` frees folios back to buddy allocator

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
- `lruvec_add_folio` adds a page to one of the lists
  - it actually adds the folio containing the page to the list

## kswapd

- kswapd thread calls `kswapd_shrink_node` to reclaim pages from various
  sources to the buddy allocator
