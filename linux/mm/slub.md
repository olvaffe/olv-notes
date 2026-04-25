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
