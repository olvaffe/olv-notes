# Kernel vmstat

## Overview

- hierarchy
  - x86 `x86_numa_init` or arm `arch_numa_init` inits numa
    - `alloc_node_data` allocs a per-node `pg_data_t`
  - `free_area_init` inits per-node `pg_data_t`
    - `free_area_init_core` inits `pgdat->node_zones`
- there are 5 categories
  - `vm_stat_item`
  - `node_stat_item`
  - `zone_stat_item`
  - `numa_stat_item`
  - `vm_event_item`
- `vm_stat_item`
  - storage
    - global `nr_memmap_boot_pages`
    - global `nr_memmap_pages`
  - these modify global storage directly
    - `memmap_boot_pages_add`
    - `memmap_pages_add`
  - getters
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
  - getters
    - `node_page_state` and `node_page_state_pages` read per-node storage
    - `global_node_page_state` and `global_node_page_state_pages` read global
      storage
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
  - getters
    - `zone_page_state` reads per-zone storage
    - `global_zone_page_state` reads global storage
    - `sum_zone_node_page_state` sums per-zone stats for a node
    - `zone_page_state_snapshot` sums per-zone stat and per-zone per-cpu stats
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
  - getters
    - `zone_numa_event_state` reads per-zone storage
    - `global_numa_event_state` reads global storage
    - `sum_zone_numa_event_state` sums per-zone stats for a node
- `vm_event_item`
  - storage
    - per-cpu `vm_event_states`
  - these modify per-cpu storage directly
    - `count_vm_event`
    - `count_vm_events`
  - getters
    - `all_vm_events` sums all cpus on the fly

## Stat Folding

- kernel makes decisions based on `node_stat_item` and `zone_stat_item`
  - for fast reading, they are folded
  - on the other hand, `vm_event_item` is consumed by userspace via
    `/proc/vmstat` and is summed on the fly
- `node_stat_item`
  - when `pgdat->per_cpu_nodestats` is updated, if `vm_node_stat_diff` exceeds
    `stat_threshold`, it calls `node_page_state_add` to fold the stat to
    global and per-node storage
  - when a cpu goes offline, `cpu_vm_stats_fold` folds
  - when nohz stops ticking, `quiet_vmstat` calls `refresh_cpu_vm_stats` to
    fold
  - every `sysctl_stat_interval` (1s), `vmstat_update` calls
    `refresh_cpu_vm_stats` to fold
- `zone_stat_item`
  - when `zone->per_cpu_zonestat` is updated, if `vm_stat_diff` exceeds
    `stat_threshold`, it calls `zone_page_state_add` to fold the stat to
    global and per-node storage
  - same as in `node_stat_item`, `cpu_vm_stats_fold`, `quiet_vmstat`, and
    `vmstat_update` fold too

## `/proc/vmstat`

- `/proc/vmstat` prints all global raw stat items
- `vmstat_start` collects the global raw stat items in order
  - it calls `global_zone_page_state` to collect global `zone_stat_item`
  - it calls `global_numa_event_state` to collect global `numa_stat_item`
  - it calls `global_node_page_state_pages` to collect global `node_stat_item`
  - it collects global `vm_stat_item`
  - it calls `all_vm_events` to collect global `vm_event_item`

## `vm_stat_item`

- `NR_DIRTY_THRESHOLD` and `NR_DIRTY_BG_THRESHOLD`
  - `global_dirty_limits` computes the thresholds on the fly
    - `global_dirtyable_memory` returns number of pages that can be dirty
      - it sums `NR_FREE_PAGES`, `NR_INACTIVE_FILE`, and `NR_ACTIVE_FILE`
      - that is, free pages in buddy, or pages allocated by page cache
    - threasholds are derived from `vm_dirty_ratio` (20%) or
      `dirty_background_ratio` (10%)
- `NR_MEMMAP_PAGES`
  - these are pages reserved for `struct page` array after boot, as a result
    of memory hotplug, etc.
- `NR_MEMMAP_BOOT_PAGES`
  - these are pages reserved for `struct page` array during boot
  - `sparse_init_nid` calls `memmap_boot_pages_add` to increment the counter
    for each section

## `node_stat_item`

- `NR_LRU_BASE`
  - this is the base enum for lru lists
    - `NR_INACTIVE_ANON`
    - `NR_ACTIVE_ANON`
    - `NR_INACTIVE_FILE`
    - `NR_ACTIVE_FILE`
    - `NR_UNEVICTABLE`
  - as page caches allocate/free pages, or as pages are moved between lru
    lists, `update_lru_size` updates page counts on each list
- `NR_SLAB_RECLAIMABLE_B` and `NR_SLAB_UNRECLAIMABLE_B`
  - slab calls `account_slab` to update the counters
- `NR_ISOLATED_ANON` and `NR_ISOLATED_FILE`
  - during reclaim, `shrink_inactive_list` isolates pages and updates the
    counters
- `WORKINGSET_*`
- `NR_ANON_MAPPED`, `NR_FILE_MAPPED`, `NR_FILE_PMDMAPPED`, `NR_ANON_THPS`, and
  `NR_SHMEM_PMDMAPPED`
  - as pages are added to rmap, these counters are updated
- `NR_FILE_PAGES`
  - as pages are allocated for page cache, shmem, swap cache, this counter is
    updated
- `NR_FILE_DIRTY`
  - as pages are marked dirty, `folio_account_dirtied` updates the counter
- `NR_WRITEBACK`
  - between `__folio_start_writeback` and `__folio_end_writeback`
- `NR_SHMEM` and `NR_SHMEM_THPS`
  - as pages are allocated for shmem, these are updated
- `NR_FILE_THPS`
  - as pages are allocated for page cache, this counter is updated
- `NR_VMSCAN_WRITE`
  - reclaim `writeout`
- `NR_VMSCAN_IMMEDIATE`
  - pages prioritized for reclaim after writeback
- `NR_DIRTIED`
  - pages dirtied
- `NR_WRITTEN`
  - pages written back
- `NR_THROTTLED_WRITTEN`
  - pages written back while reclaim is throttled
- `NR_KERNEL_MISC_RECLAIMABLE`
  - unused
- `NR_FOLL_PIN_ACQUIRED` and `NR_FOLL_PIN_RELEASED`
  - gup
- `NR_KERNEL_STACK_KB`
  - every task allocates `THREAD_SIZE` for kernel stack
- `NR_PAGETABLE`
  - number of pages used for pgtables
- `NR_SECONDARY_PAGETABLE`
  - number of pages used for nested or iommu pgtables
- `NR_IOMMU_PAGES`
  - number of pages used for iommu pgtables
- `NR_SWAPCACHE`
  - number of pages in swap cache
- `PGPROMOTE_*`
  - pages promoted to a faster memory tier
- `PGDEMOTE_*`
  - pages demoted to a slower memory tier
- `NR_HUGETLB`
  - explicit huge pages
- `NR_BALLOON_PAGES`
  - pages owned by balloon (e.g., kvm guest balloon driver)
- `NR_KERNEL_FILE_PAGES`
  - pages for page cache of fake kernel file

## `zone_stat_item`

- `NR_FREE_PAGES`
  - as pages are allocated from or freed to the buddy allocator, it calls
    `account_freepages` to count free pages
- `NR_FREE_PAGES_BLOCKS`
- `NR_ZONE_LRU_BASE`
  - this is the base enum for lru lists
    - `NR_ZONE_INACTIVE_ANON`
    - `NR_ZONE_ACTIVE_ANON`
    - `NR_ZONE_INACTIVE_FILE`
    - `NR_ZONE_ACTIVE_FILE`
    - `NR_ZONE_UNEVICTABLE`
  - `update_lru_size` updates node stats and zone stats at the same time. See
    `NR_LRU_BASE`.
- `NR_ZONE_WRITE_PENDING`
  - as pages are dirtied, `folio_account_dirtied` updates the count
- `NR_MLOCK`
  - as pages are `mlock`ed, `mlock_folio` updates the count
- `NR_ZSPAGES`
  - zram/zswap calls `zs_malloc` which updates the count
- `NR_FREE_CMA_PAGES`
  - cma calls `init_cma_reserved_pageblock` to free cma pages to buddy, and
    uses buddy to manage cma pages

## `numa_stat_item`

- buddy allocator calls `zone_statistics` to update the stats
- it counts whether a page is allocated from the preferred numa node or not

## `vm_event_item`

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
