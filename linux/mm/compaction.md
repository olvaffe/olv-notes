# Kernel Memory Compaction

## Overview

- high-level operations of `compact_zone`
  - `isolate_migratepages` scans movable lru folios from the beginning
    of the zone, isolates them from lru lists, and adds them to
    `cc->migratepages`
  - `migrate_pages` migrates `cc->migratepages`, such that there are more
    contiguous blocks at the beginning of the zone
    - `isolate_freepages` allocs dst folios from the end of the zone and adds
      them to `cc->freepages`
    - on errors, `putback_movable_pages` adds `cc->migratepages` back to lru
      lists
  - `release_free_list` frees remaining `cc->freepages`
- direct compaction
  - `__alloc_pages_slowpath -> __alloc_pages_direct_compact -> try_to_compact_pages`
  - `compact_zone_order` calls `compact_zone` on all allowed zones
- background compaction
  - `wakeup_kcompactd` is called from `wakeup_kswapd`, `kswapd_try_to_sleep`,
    etc. to wake up kcompactd
  - `kcompactd` calls `kcompactd_do_work`
    - it calls `compact_zone` on each zone
