Kernel vmalloc
=============

## vmalloc area

- `get_vm_area` finds an unused area in `[VMALLOC_START, VMALLOC_END]`
  - it allocates a `vmap_area` and a `vm_struct`
  - `vmap_area` is used to find an unused area and is added to
    `vmap_area_root`
  - `vm_struct` is returned
- `vmap_range_noflush` maps the specified va to the specified pa
  - `addr` should be from `get_vm_area` 
  - `pgd_offset_k` returns a pointer to the pgd entry
    - it uses `init_mm` whose `pgd` is `swapper_pg_dir`
  - it allocates page tables and updates entries at all levels
- this can map physically uncontiguous pages to logically contiguous addresses
- this can also be used to mmap physical mmio addresses to logical addresses

## `vmalloc`

- `__get_vm_area_node` finds an unused area
- `__vmalloc_area_node`
  - `vm_area_alloc_pages` allocates pages from the buddy allocator
  - `vmap_pages_range` calls `vmap_range_noflush` to map the allocated pages

## `vmap`

- `get_vm_area_caller` finds an unused area
- `vmap_pages_range` maps the pages into the area

## `ioremap`

- if `CONFIG_GENERIC_IOREMAP`, `ioremap` calls the generic `ioremap_prot`
- `get_vm_area_caller` finds an unused area
- `ioremap_page_range` maps the mmio addrs into the area
