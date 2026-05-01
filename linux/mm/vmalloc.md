# Kernel vmalloc

## Initialization

- `mm_core_init` calls `vmalloc_init` to init vmalloc
- `vmap_init_nodes` inits `vmap_nodes` array
  - there is a node for each cpu
- `vmap_init_free_space`
  - it allocs a `vmap_area` for `[1, ULONG_MAX]`
  - `insert_vmap_area_augment` adds the area to free map area
- `vmap_node_shrinker` is a shrinker

## VA Allocator

- `free_vmap_area_root` tracks free vas
  - `free_vmap_area_list` is for O(1) prev/next
- `vmap_nodes` tracks allocated vas
  - there are multiple nodes to reduce lock contention
- `alloc_vmap_area` allocs a va
  - `node_alloc` tries to use previous va
    - it falls back to `kmem_cache_alloc_node`
  - `__alloc_vmap_area` walks `free_vmap_area_root` and carves a range for the
    va
  - `insert_vmap_area` adds the va to its node
- `free_vmap_area` frees a va
  - `unlink_va` removes the va from its node
  - `merge_or_add_vmap_area_augment` adds the va back to `free_vmap_area_root`
- `find_vmap_area` looks up the va containing the addr
- `find_unlink_vmap_area` looks up the va containing the addr and removes it
  from its node

## Page Table Manipulation

- `vmap_range_noflush` maps pa to va
  - `pgd_offset_k` returns the pgd entry for the va in `init_mm`
  - `p4d_alloc_track` returns the p4d entry for the va
    - it allocs the p4d table and updates the pgd entry on demand
  - `pud_alloc_track` returns the pud entry for the va
    - it allocs the pud table and updates the p4d entry on demand
  - `pmd_alloc_track` returns the pmd entry for the va
    - it allocs the pmd table and updates the pud entry on demand
  - `pte_alloc_kernel_track` returns the pte entry for the va
    - it allocs the pte table and updates the pmd entry on demand
  - `set_pte_at` updates the pte entry to point to pa
- `vunmap_range_noflush` unmaps pa from va
  - `pgd_offset_k` returns the pgd entry for the va in `init_mm`
  - `p4d_offset` returns the p4d entry for the va
  - `pud_offset` returns the pud entry for the va
  - `pmd_offset` returns the pmd entry for the va
  - `pte_offset_kernel` returns the pte entry for the va
  - `ptep_get_and_clear` clears the pte entry

## `vm_struct`

- `vm_struct` is a thin wrapper for `vmap_area`
- `get_vm_area` creates a `vm_struct`
  - it allocs the struct
  - `alloc_vmap_area` creates a `vmap_area`
    - `va->vm` points to the vm
    - `vm->addr` and `vm->size` point to the va region
- after vm creation, the caller
  - might set `vm->pages`
    - for vmalloc, `vm->pages` is the allocated pages
    - for vmap, `vm->pages` is NULL unless `VM_MAP_PUT_PAGES`
  - might call `vmap_pages_range` to map pages
- `find_vm_area` looks up a vm from addr
  - `find_vmap_area` looks up the va
  - it returns `va->vm`
- `remove_vm_area` isolates a vm for freeing
  - `find_unlink_vmap_area` looks up and isolates the va
  - `free_unmap_vmap_area` zaps pgtable and frees the va
    - `vunmap_range_noflush` zaps pgtable
    - `free_vmap_area_noflush` lazy-frees the va
      - it may schedule `drain_vmap_area_work`
  - it returns the vm
- `free_vm_area` calls `remove_vm_area` and frees the vm

## `vmap` and `vunmap`

- `vmap` maps physically non-contiguous pages to logically contiguous range
  - `__get_vm_area_node` creates a `vm_struct`
  - `vmap_pages_range` sets up pgtable
    - `flush_cache_vmap` seems to be nop
- `vunmap` unmaps a range
  - `remove_vm_area` isolates the vm for freeing
  - it frees the vm

## `vmalloc` and `vfree`

- `vmalloc` allocs logically contiguous buffer
  - `__get_vm_area_node` creates a `vm_struct`
  - `__vmalloc_area_node`
    - `vm_area_alloc_pages` allocs pages
    - `__vmap_pages_range` sets up pgtable
- `vfree` a buffer
  - `remove_vm_area` isolates the vm for freeing
  - `__free_page` frees each page
  - it frees the vm

## `generic_ioremap_prot` and `generic_iounmap`

- `generic_ioremap_prot` maps a region for mmio
  - `__get_vm_area_caller` creates a `vm_struct`
  - `area->phys_addr` is set to the pa
  - `ioremap_page_range` sets up pgtable
    - `find_vm_area` looks up the vm
    - `vmap_page_range` sets up pgtable
- `generic_iounmap` simply calls `vunmap`
- on arm, `CONFIG_GENERIC_IOREMAP` is enabled
  - `ioremap` calls `generic_ioremap_prot` with `PROT_DEVICE_nGnRE`
  - `iounmap` calls `generic_iounmap`
- on x86,
  - `ioremap`
    - `memtype_reserve`
    - `get_vm_area_caller` creates a `vm_struct`
    - `memtype_kernel_map_sync`
    - `ioremap_page_range` sets up pgtable
  - `iounmap`
    - `find_vm_area` looks up the vm
    - `memtype_free`
    - `remove_vm_area` isolates the vm for freeing
    - it frees the vm
