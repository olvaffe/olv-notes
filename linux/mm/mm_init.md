# Linux MM Init

## PID 0: `start_kernel`

- x86 `setup_arch`
  - `e820__memory_setup`
    - `e820__memory_setup_default` typically has 3 or 4 ram entries
      - `[0, ~640KB]`
      - `[1MB, ~2GB]`
      - `[4GB, ~REMAINING_RAM]`
  - `trim_bios_range` updates e820 to reserve first 4KB, etc.
  - `max_pfn` is set to `e820__end_of_ram_pfn`, which is last page
  - `max_low_pfn` is set to `e820__end_of_low_ram_pfn`, which is last page before 4GB
    - will be updated later
  - `e820__memblock_setup` calls `memblock_add` for each e820 ram entry
  - `init_mem_mapping` sets up page table for kernel linear map
    - it also updates `max_low_pfn` to `max_pfn`
  - `initmem_init` calls `x86_numa_init` to init numa
    - without `CONFIG_NUMA`, all memblock ranges belong to node 0
    - `numa_init`
      - `dummy_numa_init` fakes node 0 for all physical ram
      - `alloc_node_data` allocs `pg_data_t` and inits `NODE_DATA(nid)`
  - `x86_init.paging.pagetable_init` is `paging_init`
- arm `setup_arch -> early_init_dt_scan -> early_init_dt_scan_nodes`
  - `early_init_dt_scan_memory` parses nodes with `device_type = "memory"`
    - `early_init_dt_add_memory_arch` adds a memblock
  - `early_init_dt_check_for_usable_mem_range` parses extra ranges
- `mm_core_init_early` calls `free_area_init`
  - `arch_zone_limits_init` gets zone limits
    - `ZONE_DMA` is capped by `MAX_DMA_PFN` (16MB)
    - `ZONE_DMA32` is capped by `MAX_DMA32_PFN` (4GB)
    - `ZONE_NORMAL` is capped by `max_pfn`
  - `sparse_init` inits `mem_section` and `vmemmap`
    - `vmemmap` is the `struct page` array
  - zones are initialized according to zone limits
  - `for_each_mem_pfn_range` loops over memblock ranges
  - `for_each_node` loops over each node (only node 0)
    - `free_area_init_node` fully inits `NODE_DATA(nid)`, which is `pg_data_t`
    - `node_set_state` adds the nid to `N_MEMORY` and `N_NORMAL_MEMORY`
  - `calc_nr_kernel_pages` inits `nr_kernel_pages`
    - `for_each_free_mem_range` loops over memblock ranges that are not reserved
  - `memmap_init` inits memmap (`struct page` array)
    - `__init_single_page` inits a single `struct page`
    - `init_unavailable_range` sets `PG_reserved` on holes
      - `sparse_init` allocates `struct page` array in sections
      - each section covers 128MB
      - if a memblock range is not aligned, there are `struct page`s that do
        not correspond to physical ram
  - `set_high_memory` inits `high_memory` to point to end of ram
- `mm_core_init`
  - `build_all_zonelists` builds zone lists
    - `check_highest_zone` updates `policy_zone` to `ZONE_NORMAL`
  - `report_meminit` reports if pages are zeroed on alloc/free
    - `mem auto-init: stack:all(zero), heap alloc:on, heap free:off`
  - `memblock_free_all` frees pages from memblock to buddy

## PID 1: `kernel_init`

- `init_mm_internals` inits vmstats
- `page_alloc_init_late`
  - `mem_init_print_info` summaries mem info
- initcalls
- `free_initmem` frees kernel sections needed only for init
- exec to userspace

## memory model

- arch configs
  - common between x86 and arm64 by default
    - `CONFIG_NUMA=y`
    - `CONFIG_NUMA_BALANCING=y`
    - `CONFIG_SPARSEMEM=y`
    - `CONFIG_SPARSEMEM_EXTREME=y`
    - `CONFIG_SPARSEMEM_VMEMMAP=y`
    - `CONFIG_ZONE_DMA=y`
    - `CONFIG_ZONE_DMA32=y`
  - x86 by default
    - `CONFIG_NR_CPUS=64`
    - `CONFIG_PGTABLE_LEVELS=5`
    - `CONFIG_NODES_SHIFT=6`
  - arm64 by default
    - `CONFIG_NR_CPUS=256`
    - `CONFIG_PGTABLE_LEVELS=5`
    - `CONFIG_NODES_SHIFT=4`
    - `CONFIG_ARM64_4K_PAGES=y`
    - `CONFIG_ARM64_PA_BITS_52=y`
    - `CONFIG_ARM64_VA_BITS_52=y`
- 5-level paging supports 52-bit pa
  - 52-bit pa is divided into 27-bit sections
    - `MAX_PHYSMEM_BITS` is 52
    - `SECTION_SIZE_BITS` is 27
    - `PAGES_PER_SECTION` is 32K
  - there are 32M sections
    - `SECTIONS_SHIFT` is 25
    - `NR_MEM_SECTIONS` is 32M
  - each root is 4KB and can hold 256 `mem_section`
    - `SECTIONS_PER_ROOT` is 256
  - we need 128K roots
    - `NR_SECTION_ROOTS` is 128K
  - `mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT]` array is sparsely
    allocated
- page flags layout
  - page flags use slightly less than 32 (`__NR_PAGEFLAGS`) bits
  - zone is in page flags and takes 2 or 3 (`ZONES_WIDTH`) bits
  - section is not in page flags and takes 0 (`SECTIONS_WIDTH`) bits
  - node is in page flags and takes 4 or 6 (`NODES_WIDTH`) bits
  - last cpu id is in page flags and takes 14 or 16 (`LAST_CPUPID_SHIFT`) bits
  - as a result, from the top bits to the bottom bits, we have
    - node at the top bits
    - zone
    - last cpupid
    - page flags at the bottom bits

## `sparse_init`

- assume x86 with 32GB of physical ram
  - e820 typically reports ~3 `System RAM` entries
    - `[0, ~640KB]`, or after `trim_bios_range`, `[4KB, ~640KB]`
    - `[1MB, ~2GB]`
    - `[4GB, ~32GB]`
  - `memblock_add` adds a range for each entry
- `start_kernel -> mm_core_init_early -> free_area_init -> sparse_init`
  - `memblocks_present`
    - it allocates `mem_section` array
      - each `mem_section[i]` is a root and will be allocated on demand
      - there are 128K roots thus the array is 1MB in size
    - `memblocks_present` marks each of ~3 e820 entries as present
      - `sparse_index_init` allocs each root (`mem_section[i]`) on demand
        - it allocs an array of 256 sections, taking up 4KB
      - with 32GB of system ram plus gaps/mmios, we need slightly more than
        256 sections thus two roots
  - `sparse_init_nid`
    - `pnum_begin` is the first present section
      - because first 640KB is ram, this is 0
    - `pnum_end` is the last present section
      - with 32GB ram + gaps/mmios, this is slightly more than 256
    - `map_count` is the number of present sections
      - roughly 256
    - `sparse_buffer_init` preallocs `sparsemap_buf`, array of all `struct page`
      - the size is `map_count * sizeof(struct page) * PAGES_PER_SECTION`,
        roughly 512MB
    - `__populate_section_memmap` sets up pgtable such that `vmemmap[pfn]`
      maps to the corresponding `struct page` in `sparsemap_buf` array
    - `sparse_init_early_section` inits a `mem_section`
      - `ms->section_mem_map` points to the page array (plus flags)
- `__populate_section_memmap`
  - the goal is to set up pgtable such that `vmemmap[pfn]` maps to the
    corresponding `struct page` preallocated in `sparsemap_buf`
    - `page_to_pfn` and `pfn_to_page` become trivial
      - `pfn_to_page` expands `vmemmap + pfn`
      - `page_to_pfn` expands `page - vmemmap`
    - note that `vmemmap` has holes while `sparsemap_buf` does not
      - because pa of physical ram has roles and pfn is simply `PHYS_PFN(pa)`
  - arch-specific `vmemmap_populate` calls generic
    `vmemmap_populate_hugepages` to set up pgtable for a section of
    `PAGES_PER_SECTION` pages
    - `[start, end]` is va within `vmemmap`
    - `pgd`, `p4d`, `pud`, and `pmd` are the pgtable entries for the va
    - `vmemmap_alloc_block_buf` returns va of corresponding `struct page`s in
      `sparsemap_buf`
    - `vmemmap_set_pmd` updates pmd entry to point to the pa of the
      corresonding `struct page`s in `sparsemap_buf`
  - on x86, `vmemmap` is effectively `__VMEMMAP_BASE_L5`
    (0xffd4000000000000UL)
