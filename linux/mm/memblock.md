# Kernel memblock

## Overview

- `memblock` is a boot-time memory manager
  - it is used before the page allocator is initialized
  - `memory` contains regions of all physical ram
  - `reserved` contains regions of in-use physical ram
- init
  - `memblock_add` adds a fw-reported physical ram region to `memory`
  - `memblock_reserve` adds reserved (in-use) ram to `reserved`
- alloc/free
  - `memblock_alloc` allocs a range
    - `memblock_find_in_range_node` find a range that is in `memory` but not in
      `reserved`
    - `__memblock_reserve` adds the range to `reserved`
  - `memblock_free` frees a range
    - `memblock_remove_range` removes the range from `reserved`
- cleanup
  - `memblock_free_all` hands over non-reserved regions to the page allocator
- loop macros
  - `for_each_mem_region` loops over regions in `memory`
    - `for_each_mem_range` returns begin/end pa of the regions
    - `for_each_mem_pfn_range` returns begin/end pfn of the regions
  - `for_each_reserved_mem_region` loops over regions in `reserved`
    - `for_each_reserved_mem_range` returns begin/end pa of the regions
  - `for_each_free_mem_range` loops over `memory` and excludes `reserved`
- queries
  - `memblock_phys_mem_size` returns total size of `memory`
  - `memblock_reserved_size` returns total size of `reserved`
  - `memblock_start_of_DRAM` returns the base addr of the first region of `memory`
  - `memblock_end_of_DRAM` returns the end addr of the last region of `memory`
  - `memblock_is_memory` returns true if a pa is in `memory`
  - `memblock_is_reserved` returns true if a pa is in `reserved`
- flags
  - `MEMBLOCK_NOMAP` omits the region from the kernel linear mapping
