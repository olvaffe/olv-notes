# Linux slub

## Overview

- page allocator (`page_alloc.c`), aka buddy allocator, allocates physically
  contiguous pages
- slub allocator, aka slab allocator, sub-allocates from the page allocator
  - allocations are physically contiguous if they cross the page boundary
    - because `kmalloc` makes use of the kernel linear mapping where all
      physical pages are mapped linearly
- kmem cache is optimized for same-size allocations
  - `kmem_cache_create` creates a `kmem_cache`
  - `kmem_cache_alloc` allocates an object

## `kmalloc`

- if the size is greater than `KMALLOC_MAX_CACHE_SIZE` (two pages),
  `__kmalloc_large_node_noprof` calls `alloc_frozen_pages_noprof` directly
- `kmalloc_slab` selects a `kmem_cache`
  - it picks from `kmalloc_caches` first depending on the flags
    - `KMALLOC_NORMAL`, `KMALLOC_RECLAIM`, `KMALLOC_DMA`, `KMALLOC_CGROUP`
  - it then picks a `kmem_cache` depending on the size
- `slab_alloc_node` allocs from the selected `kmem_cache`
  - `slab_pre_alloc_hook` is almost nop
    - `kernel_init_freeable` sets `gfp_allowed_mask` from `GFP_BOOT_MASK` to
      `__GFP_BITS_MASK`
  - `alloc_from_pcs` is the fast path
    - if `s->cpu_sheaves->main` is empty, `__pcs_replace_empty_main` tries to
      refill the sheaf by calling `refill_sheaf`
    - it then pops from `s->cpu_sheaves->main->objects` if any
  - `__slab_alloc_node` is the medium/slow path
    - `get_from_partial` tries to suballoc from existing slabs first
      - it scans `n->partial`, the list of partially allocated slabs
        - `__slab_update_freelist` updates `slab->freelist_counters`
          atomically
    - `new_slab` allocates a new slab and is the slowest path
      - note that `struct slab` overlays `struct page`
      - `slab->freelist` is the singly-linked list of free objects
    - `alloc_from_new_slab` sub-allocs from the new slab
      - it pops an object from `slab->freelist`
  - `slab_want_init_on_alloc` returns true if `__GFP_ZERO` or
    `CONFIG_INIT_ON_ALLOC_DEFAULT_ON`
  - `slab_post_alloc_hook` post-processes the newly allocated obj
    - it zeros the object if requested
    - `memcg_slab_post_alloc_hook` charges to memcg
