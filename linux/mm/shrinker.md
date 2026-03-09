Kernel Shrinker
===============

## Overview

- when memory is low, `try_to_free_pages` or `kswapd` invokes `shrink_node` to
  reclaim memory
  - `shrink_node` calls `shrink_lruvec` to shrink lru and calls `shrink_slab`
    to shrink shrinkers
- alternatively, `echo 2 > /proc/sys/vm/drop_caches` calls `drop_slab`
  - `drop_slab_node` also calls `shrink_slab`
- `shrink_slab` calls `do_shrink_slab` on each shrinker
