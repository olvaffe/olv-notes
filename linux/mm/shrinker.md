# Kernel Shrinker

## Overview

- who calls `shrink_node`
  - direct light reclaim
    - `node_reclaim -> __node_reclaim -> shrink_node`
  - direct heavy reclaim
    - `try_to_free_pages -> do_try_to_free_pages -> shrink_zones -> shrink_node`
  - background reclaim
    - `kswapd -> balance_pgdat -> kswapd_shrink_node -> shrink_node`
  - more
- how does `shrink_node` call `shrink_slab`
  - if mglru, `lru_gen_shrink_node -> (shrink_many) -> shrink_one -> try_to_shrink_lruvec+shrink_slab`
  - else, `shrink_node_memcgs -> shrink_lruvec+shrink_slab`
- what else calls `shrink_slab`
  - `echo 2 > /proc/sys/vm/drop_caches`
    - `drop_caches_sysctl_handler -> drop_slab -> drop_slab_node -> shrink_slab`
- what does `shrink_slab` do
  - it calls `do_shrink_slab` on each shrinker

## `do_shrink_slab`

- it calls `shrinker->count_objects` to count potentially freeable objects
  - an "object" can be anything, often a page
  - `SHRINK_EMPTY` means no potentially freeable objects
    - while 0 means to skip this shrinker
    - the two are the same unless this is called from `shrink_slab_memcg` for memcg
- `xchg_nr_deferred` gets and clears `shrinker->nr_deferred`
  - this is objects-to-scan minus objects-scanned from the prior calls
- it computes `delta` and `total_scan`
  - `priority` oftens goes from `DEF_PRIORITY` (12) to 0
    - the caller shrinks all shrinkers at high prior values first, such that
      all shrinkers contribute
  - `delta` is the number of additional objects to scan by this call
    - `shrinker->seeks` defaults to `DEFAULT_SEEKS` (2), indicating how many
      IOs are needed to recreate an object if freed
  - `total_scan` is the number of total objects to scan
    - it is roughly `(shrinker->nr_deferred >> priority) + delta`
- it calls `shrinker->scan_objects` in a loop to free objects
  - it sets `sc->nr_to_scan` and `sc->nr_scanned` to `min(batch_size, total_scan)`
    - `batch_size` oftens defaults to `SHRINK_BATCH` (128)
  - each `shrinker->scan_objects` call
    - scans at most `sc->nr_to_scan` objects and updates `sc->nr_scanned`
    - returns number of freed objects, or `SHRINK_STOP` to bail
  - it breaks when `scanned` is close to original `total_scan`
- `add_nr_deferred` updates `shrinker->nr_deferred` roughly to the number of
  objects that are to-be-scanned but are not scanned
- tracepoints
  - `vmscan/mm_shrink_slab_start`
    - `objects to shrink` is `shrinker->nr_deferred`
    - `cache items` is `shrinker->count_objects`
    - `delta` and `total_scan` are original calculated values
  - `vmscan/trace_mm_shrink_slab_end`
    - `unused scan count` is original `shrinker->nr_deferred`
    - `new scan count` is updated `shrinker->nr_deferred`
    - `total_scan` is original `total_scan` minus `scanned`
    - `last shrinker return val` is number of freed objects

## `CONFIG_SHRINKER_DEBUG`

- `shrinker_debugfs_add` creates `/sys/kernel/debug/shrinker/<name>` for each
  shrinker
- `shrinker_debugfs_count_show` calls `shrinker->count_objects` directly
  - the output format is `<memcg-id> <per-node-freeable-object-counts>`
- `shrinker_debugfs_scan_write` calls `shrinker->scan_objects` directly
  - the input format is `<memcg-id-or-zero> <node-id> <number-of-objects-to-scan>`
