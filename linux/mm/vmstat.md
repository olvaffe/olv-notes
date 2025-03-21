Kernel vmstat
=============

## `/proc/vmstat`

- `zone_stat_item`
 -  `vm_zone_stat` is the counters
- `numa_stat_item`
  - `vm_numa_event` is the counters
- `node_stat_item`
 -  `vm_node_stat` is the counters
- `vm_stat_item`
  - `global_dirty_limits`
  - `nr_memmap_pages`
  - `nr_memmap_boot_pages`
- `vm_event_item`
  - `vm_event_states` is the per-cpu counters
    - `all_vm_events` sums them up
  - `PGPGIN` / `pgpgin`
    - when `submit_bio` submits a block io, it counts `PGPGIN` for sectors
      read and `PGPGOUT` for sectors written
  - `PSWPIN` / `pswpin`
    - when `swap_read_folio` swaps a page in from the block device, it counts
      toward `PSWPIN`
  - `PSWPOUT` / `pswpout`
    - when `swap_writepage` swaps a page out to the block device, it counts
      toward `PSWPOUT`
  - `PGSTEAL_KSWAPD` / `pgsteal_kswapd`
    - `kswapd_shrink_node` shrinks a node to reclaim pages from various
      sources
      - `shrink_inactive_list` counts `PGSTEAL_KSWAPD` for reclaimed pages
  - `PGSCAN_KSWAPD` / `pgscan_kswapd`
    - `kswapd_shrink_node` shrinks a node to reclaim pages from various
      sources
      - `shrink_inactive_list` counts `PGSCAN_KSWAPD` for scanned pages
  - `PGSCAN_KSWAPD` / `pgscan_kswapd`
    - `kswapd_shrink_node` shrinks a node to reclaim pages from various
      sources
      - `shrink_inactive_list` counts `PGSCAN_KSWAPD` for scanned pages
  - `SWAP_RA` / `swap_ra`
    - when `do_swap_page` swaps in a page, `swapin_readahead` reads more pages
      than needed in the hope that they are going to be needed soon
      - `swap_vma_readahead` counts `SWAP_RA`
  - `SWAP_RA_HIT` / `swap_ra_hit`
    - when `do_swap_page` swaps in a page, `swap_cache_get_folio` looks up the
      page in the swap cache first
      - it counts `SWAP_RA_HIT` if the page is in the cache thanks to
        readahead
  - `SWPIN_ZERO` / `swpin_zero`
    - when `swap_read_folio` swaps pages in, if the pages are known zero, it
      can alloc and zero pages without reading back from the block device
      - `swap_read_folio_zeromap` counts `SWPIN_ZERO`
  - `SWPOUT_ZERO` / `swpout_zero`
    - when `swap_writepage` swaps pages out, if the pages are known zero, it
      can mark so without writing them to the block device
      - `swap_zeromap_folio_set` counts `SWPIN_ZERO`
