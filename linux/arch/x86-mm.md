X86 MM
======

## config

- `64BIT`
- `X86_64`
- `MTRR`
- `X86_PAT`
- `DYNAMIC_MEMORY_LAYOUT`
- `SPARSEMEM_VMEMMAP`
- `SPARSEMEM_EXTREME`
- `SPARSEMEM`
- `PHYS_ADDR_T_64BIT`
- `NEED_SG_DMA_LENGTH`
- `NEED_DMA_MAP_STATE`
- `ARCH_DMA_ADDR_T_64BIT`
- `SLUB`
- `HAVE_MEMBLOCK_NODE_MAP`
- `HAVE_ARCH_HUGE_VMAP`
- `SWIOTLB`
- `ARCH_HAS_PTE_SPECIAL`
- no `X86_5LEVEL`
- no `HAVE_MEMBLOCK_PHYS_MAP`
- no `DEFERRED_STRUCT_PAGE_INIT`
- no `ARCH_HAS_SYNC_DMA_FOR_DEVICE`
- no 32-bit specific tricks
  - no `X86_32`
  - no `HIGHMEM4G`
  - no `VMSPLIT_3G`
  - no `PAGE_OFFSET`
  - no `PAE`
  - no `HIGHPTE`

## init

- get physical memory map from BIOS
  - `e820__memory_setup` calls `e820__memory_setup_default`
  - the table is copied from `boot_params` to `e820_table`
    - `boot_params` was set up with `detect_memory_e820`
- update e820 table to reserve setup data
  - `e820__reserve_setup_data` finds setup data from `boot_params` and updates
    `e820_table`
- prepare page table storage
  - the storage uses the brk data section of the kernel binary
  - `early_alloc_pgt_buf` claims a area from the brk section
  - `alloc_low_pages` will use the area first before using memblock
- setup memblock allocator according to e820
  - `e820__memblock_setup`
  - `memblock_add` adds a usable RAM range to memblock
  - `memblock_reserve` marks a usable RAM range reserved
  - regions containing BIOS or EFI data should be marked reserved
  - an allocation from memblock finds a usable range and mark it reserved
- setup linear map
  - `init_mem_mapping`
  - `probe_page_size_mask` checks if GB huge page is possible (yes)
  - `init_mm` is the mm, and `init_mm.pgd` is `init_top_pgt`
    - i.e., the 4KB space for pgt is in the data section
  - the storage of the reset of the page tables is allocated using
    `alloc_low_page`, which uses the area prepared by `early_alloc_pgt_buf`
- set up `struct page` array and set up zones for buddy allocator
  - `x86_init.paging.pagetable_init` points to `native_pagetable_init`
  - `sparse_init_nid` allocates the `struct page` array
    - `sparse_buffer_init` allocates the storage from memblock
  - `zone_sizes_init`
- free pages from memblock to buddy allocator
  - `memblock_free_all` called by `mem_init`
- on a machine with 32G of memory,
  - e820 reports about 31.7G usable
  - `zone_sizes_init` reports a similar amount of memory
  - `mem_init_print_info` reports a similar amount of memory
    - `get_num_physpages()` is 31.7G but `totalram_pages()` is 30.8G
    - the delta is reported as reserved and consists of
      - kernel text, data, bss, and init sections
      - `struct page` array
      - hw carved-out regions
    - free pages are limited to those in lowmem thanks to
      `CONFIG_DEFERRED_STRUCT_PAGE_INIT`
  - `free_reserved_area` frees up some reserved memory
    - init section, initrd, etc.
    - this increases `totalram_pages()`
  - `grep MemTotal /proc/meminfo` reports 31.06G
    - it reports `totalram_pages()`
- `start_kernel`
  - `setup_arch`
    - set up memblock according to e820, `e820__memblock_setup`
      - `memblock_add` adds a memory region to memblock
      - `memblock_reserve` adds a reserved region to memblock
        - a range that is in the memory regions but not in the reserved
          regions are considered available
      - an allocation finds a range in the memory regions but not in the
        reserved regions, and then adds the range to the reserved regions
    - `init_mem_mapping` initializes `PAGE_OFFSET` linear mapping
      - direct mappings to all physical memory regions; gaps between the
        memory regions are not mapped
      - va `PAGE_OFFSET + x`  is mapped to pa `x` as long as x belong to a
        physical memory region
      - space for page tables are taken from the BSS of the kernel image
    - `initmem_init` assigns all memory regions in memblock to node 0
    - `native_pagetable_init` is defined to `paging_init`
      - `sparse_memory_present_with_active_regions` calls `memory_available`
        on all memory regions in memblock
    - with the default EXTREME mode, each root has `SECTIONS_PER_ROOT`
      (256) sections and each section has `PAGES_PER_SECTION` (2^15) pages
    - it allocates 8MiB worth of memory from memblock for the roots, to
      cover the entire 46-bit of physical memory address space
    - the number of "present" sections depend on the size of the physical
      memory
      - `sparse_init` calls `sparse_mem_map_populate` (vmemmap version) for
        each present section, which allocates page tables and `struct page`
        from memblock, and set up mappings to access `struct page` from va
        `vmemmap` region
    - va `vmemmap + sizeof(struct page) * pfn` is mapped to `struct page`
      for pfn, if `pfn << PAGE_SHIFT` is an address of the phsycal memory
      - `zone_sizes_init` tells the buddy page allocator the sizes of memory
        zones
  - `mm_init`
    - `mem_init`
      - `memblock_free_all` releases all pages from memblock to the buddy page
        allocator.  Those in the reserved regions are marked `PG_reserved`.
    - `kmem_cache_init` initializes the slab allocator on top of the buddy
      allocator
    - `vmalloc_init` prepares for vmallocs/ioremaps that will use
      `VMALLOC_START` and onward (32TB)

## brk section

- Introduced in `93dbda7cbcd70a0bd1a99f39f44a9ccde8ab9040`, 2009.
  - And modified several times after.
- In the linker script, `.brk` is reserved for `*(.brk_reservation)` and
  is delimited by `__brk_base` and `__brk_limit`.  It comes just after `.bss`
  and just before `.end`.
- `RESERVE_BRK` is used to reserve brk space
  - It creates `.brk_reservation` section with the given size
  - It is used to reserve space for init page tables and dmi
- When `head_32.S` creates `default_entry`, it creats page tables in `.brk`.
  - `_brk_end` marks the location after the page tables.
  - `extend_brk` is used to alloc space in brk.  It extends `_brk_end`.
    - It is used indirectly by `dmi_alloc` in `drivers/firmware/dmi_scan.c`.
- `reserve_brk` is called to reserve brk as `reserve_early`.  It also pins the
  brk as read-only.
  - It reserves up to `_brk_end`, not to `_brk_limit`.  Therefore, it is safe to
    `RESERVE_BRK` a large (safe) region.

## Paging Table

- 4-level paging
  - 48-bit virtual address and 46-bit physical address
  - CR3 bit 51..12: PA to the 4KB-aligned PML4 (Page Map Level 4) table
    - 512 64-bit entries called PML4Es
  - VA bit 47..39: select one of the 512 PML4Es
    - each PML4E contains a PA to a PDP (page-directory-pointer) table
    - there are 512 64-bit entries called PDPEs
  - VA bit 38..30: select one of the 512 PDPEs
    - each PDPE contains a PA to a PD (page directory)
    - there are 512 64-bit entries called PDEs
  - VA bit 29..21: select one of the 512 PDEs
    - each PDE contains a PA to a page table
    - there are 512 64-bit entries called PTEs
  - VA bit 20..12: select one of the 512 PTEs
    - each PTE contains a PA to a page
  - VA bit 11..0: 12-bit offset into the page
- 5-level paging
  - 57-bit virtual address and 52-bit physical address
  - CR3 bit 51..12: PA to the 4KB-aligned PML5 (Page Map Level 5) table
  - VA bit 56..48: select one of the 512 PML5Es
    - each PML5E contains a PA to a PML4 table
    - there are 512 64-bit entries called PML5Es
- All entries at all levels have a similar (but different) format
  - bit 63..52: 12-bit for flags
    - bit 63 is NX (no execute)
    - more
  - bit 51..12: 40-bit pfn
  - bit 11..0: 12-bit for flags
    - bit 4 is page cache disabled (for the next table; or page when it is PTE)
    - bit 3 is page write through (for the next table; or page when it is PTE)
    - bit 1 is writable
    - bit 0 is present
    - more

## Page Fault

- in `idt_setup_early_pf`, `asm_exc_page_fault` is used for `X86_TRAP_PF`
- `DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF, exc_page_fault)` expands to
  `idtentry` to define `asm_exc_page_fault`
  - it is very standard as other exception vectors
  - `call error_entry`
  - `call exc_page_fault`
  - `jmp error_return`
- `DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)` defines `exc_page_fault`
  - `irqentry_enter`
  - `handle_page_fault` calls `do_user_addr_fault`
  - `irqentry_exit`
- `do_user_addr_fault`
  - `mm` is from `current->mm`
  - `mmap_read_trylock` locks the mm for read access
  - `find_vma` returns the vma for the fault address
    - `expand_stack` takes care of automatic stack grow
  - `handle_mm_fault` is the generic fault function

## Memory Initialization (old)

- x86-64 uses 4-level page table.
  - `PGDIR_SHIFT == 39`
  - `PUD_SHIFT == 30`
  - `PMD_SHIFT == 21`.
  - `PTRS_PER_PGD` is 512, `1 << (48 - 39)`.
  - `PTRS_PER_PUD` is 512, `1 << (39 - 30)`.
  - `PTRS_PER_PMD` is 512, `1 << (30 - 21)`.
  - `PTRS_PER_PTE` is 512, `1 << (21 - 12)`.
- There are three memory allocators
  - brk
  - memblock
  - e820 and `alloc_low_page`
  - buddy allocator
- In `start_kernel`, after the linux banner is printed, `setup_arch` is called.
  - It calls `e820__memory_setup` to decide physical memory maps.
  - It calls `e820__end_of_ram_pfn` to decide 
    - `max_pfn`, (pfn of the last page of the DRAM) + 1.
    - `max_arch_pfn`, `1 << MAX_PHYSMEM_BITS`
  - It calls `e820__end_of_low_ram_pfn` to decide
    - `max_low_pfn`, which is `min(max_pfn, 4GB>>12)`
  - It calls `reserve_brk` to resert brk region.
  - It calls `init_mem_mapping` to map the low memory.
    - It runs before bootmem for early direct access.
    - The first range is the first 4MB (`PMD_SIZE`), using 4K page.
    - The second range is the rest, using 4M page
    - It calls `find_early_table_space` to reserve in `e820_table_start` and
      `e820_table_end` a place to store the page tables.
      - This storage is available to others through `alloc_low_page`.  It is
        the `MAPPING_BEYOND_END` mapped in `head_32.S` 
      - This storage is reserved by `reserve_early`.
    - The mappings are set up by `kernel_physical_mapping_init`.
    - cr3 is loaded, with `swapper_pg_dir` still being the PDE table.
    - `max_low_pfn_mapped` and `max_pfn_mapped` are set to `max_low_pfn`
      if everything goes normally.
    - That is, the mapped memory goes from 8MB to whole low memory.
  - It calls `initmem_init` to set up bootmem, boot-time physical memory
    allocator.
    - `e820_register_active_regions` is called with node 0 and all physical
      memory.  All e820 ram regions are stored in `early_node_map` of buddy
      allocator.
    - `setup_bootmem_allocator` is called.  It sets `after_bootmem` to 1.
    - It `reserve_early` an area for `bootmap`.
    - Later, `early_res_to_bootmem` is called to migrate early reserved area to
      bootmem map.
  - It calls `paging_init` to allocate an array of all `struct page *` from
    bootmem.
    - All pages are initialized as reserved at this point by `memmap_init_zone`.
- Much later in `start_kernel`, `mem_init` is called
  - It calls `free_all_bootmem` to return unused pages in bootmem to the buddy
    allocator.
  - `zap_low_mappings` is called to, forget low memory mappings.  That is,
    virtual address `< PAGE_OFFSET` is no longer mapped.  This is so that one
    can only access kernel memory from kernel address space.
- An example of early reserved regions

    (8 early reservations) ==> bootmem [0000000000 - 00377fe000]
      #0 [0000000000 - 0000001000]   BIOS data page ==> [0000000000 - 0000001000]
      #1 [0000001000 - 0000002000]    EX TRAMPOLINE ==> [0000001000 - 0000002000]
      #2 [0000006000 - 0000007000]       TRAMPOLINE ==> [0000006000 - 0000007000]
      #3 [0000100000 - 00004cb180]    TEXT DATA BSS ==> [0000100000 - 00004cb180]
      #4 [00004cc000 - 00004cf000]    INIT_PG_TABLE ==> [00004cc000 - 00004cf000]
      #5 [000009fc00 - 0000100000]    BIOS reserved ==> [000009fc00 - 0000100000]
      #6 [0000007000 - 0000008000]          PGTABLE ==> [0000007000 - 0000008000]
      #7 [0000008000 - 000000f000]          BOOTMAP ==> [0000008000 - 000000f000]
  - PGTABLE is one page because only the first 4M uses 4K page.  The rest uses
    4M page and does not need to allocate memory.
