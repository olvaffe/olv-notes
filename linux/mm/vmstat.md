Kernel vmstat
=============

## Overview

- hierarchy
  - x86 `x86_numa_init` or arm `arch_numa_init` inits numa
    - `alloc_node_data` allocs a per-node `pg_data_t`
  - `free_area_init` inits per-node `pg_data_t`
    - `free_area_init_core` inits `pgdat->node_zones`
- there are 5 categories
  - `vm_stat_item`
    - `NR_DIRTY_*`
    - `NR_MEMMAP_*`
  - `node_stat_item`
    - `NR_*`
    - `WORKINGSET_*`
    - `PGPROMOTE_*`
    - `PGDEMOTE_*`
  - `zone_stat_item`
    - `NR_*`
    - `WORKINGSET_*`
    - `PGPROMOTE_*`
    - `PGDEMOTE_*`
  - `numa_stat_item`
    - `NUMA_*`
  - `vm_event_item`
    - `PG*`
    - `PGSTEAL_*`
    - `PGSCAN_*`
    - `SLABS_*`
    - `KSWAPD_*`
    - `PAGE*`
    - `DROP_*`
    - `OOM_*`
    - `NUMA_*`
    - `PGMIGRATE_*`
    - `THP_*`
    - `COMPACT*`
    - `KCOMPACTD_*`
    - `HTLB_*`
    - `CMA_*`
    - `UNEVICTABLE_*`
    - `BALLOON_*`
    - `NR_TLB_*`
    - `SWAP_*`
    - `SWP*`
    - `KSM_*`
    - `COW_*`
    - `ZSWP*`
    - `DIRECT_MAP_*`
    - `VMA_LOCK_*`
    - `KSTACK_*`
- `vm_stat_item`
  - storage
    - global `nr_memmap_boot_pages`
    - global `nr_memmap_pages`
  - these modify global storage directly
    - `memmap_boot_pages_add`
    - `memmap_pages_add`
  - `global_dirty_limits` computes dirty thresholds on the fly
- `node_stat_item`
  - storage
    - global `vm_node_stat`
    - per-node `pgdat->vm_stat`
    - per-node per-cpu `pgdat->per_cpu_nodestats[cpu].vm_node_stat_diff`
      - this appears temporary and will be folded into global and per-node
        storage
  - these modify per-node per-cpu storage directly
    - `__mod_node_page_state`
    - `__inc_node_state`
    - `__dec_node_state`
    - `mod_node_state`
    - more
  - these modify global and per-node storage directly
    - `node_page_state_add`
    - more
- `zone_stat_item`
  - storage
    - global `vm_zone_stat`
    - per-zone `zone->vm_stat`
    - per-zone per-cpu `zone->per_cpu_zonestats[cpu].vm_stat_diff`
      - this appears temporary and will be folded into global and per-zone
        storage
  - these modify per-zone per-cpu storage directly
    - `__mod_zone_page_state`
    - `__inc_zone_state`
    - `__dec_zone_state`
    - `mod_zone_state`
    - more
  - these modify global and per-zone storage directly
    - `zone_page_state_add`
    - more
- `numa_stat_item`
  - storage
    - global `vm_numa_event`
    - per-zone `zone->vm_numa_event`
    - per-zone per-cpu `zone->per_cpu_zonestats[cpu].vm_numa_event`
      - this appears temporary and will be folded into global and per-zone
        storage
  - these modify per-zone per-cpu storage directly
    - `__count_numa_event`
    - `__count_numa_events`
  - these modify global and per-zone storage directly
    - `zone_numa_event_add` modifies per-zone and global stats
- `vm_event_item`
  - storage
    - per-cpu `vm_event_states`
  - these modify per-cpu storage directly
    - `count_vm_event`
    - `count_vm_events`
  - `all_vm_events` sums all cpus on the fly

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
