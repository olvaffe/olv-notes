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
