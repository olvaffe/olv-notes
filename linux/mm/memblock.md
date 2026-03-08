Kernel memblock
===============

## Overview

- memblock is a boot-time memory manager, used before the page allocator is
  initialized
- `memblock_add` adds fw-reported physical ram to `memory`
- `memblock_reserve` adds reserved (in-use) ram to `reserved`
- `memblock_alloc` allocs a range from `memory` that does not conflict with
  `reserved`
  - it also adds the range to `reserved`
- `memblock_free_all` hands over memories to the page allocator for it to take
  over
