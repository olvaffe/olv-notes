# X86 MM

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

- 5-level paging
  - 57-bit virtual address and 52-bit physical address
  - CR3 bit 51..12: PA to the 4KB-aligned PML5 (Page Map Level 5) table
    - 512 64-bit entries called PML5Es
  - VA bit 56..48: select one of the 512 PML5Es
    - each PML5E contains a PA to a PML4 table
    - there are 512 64-bit entries called PML5Es
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
- PGD / PML5
  - `pgd_t` represents a pgd entry
    - a pgd table is page-sized and consists of 512 pgd entries
    - when `pgt_t *` points to the first entry of a table, we can treat it as
      pointing to the entire table
  - `pgd_alloc` allocates a pgd table
    - `mm->pgd` points to the table
    - when context-switch, cr3 is updated to `mm->pgd`
  - `pgd_offset` returns the pgd entry for the va
    - `&mm->pgd[(va >> 48) & 0xff]`
  - `pgd_addr_end`
  - `pgd_bad` checks if the entry has valid value
  - `pgd_clear` clears the entry value to 0
  - `pgd_clear_bad` clears the entry value to 0 after logging an error
  - `pgd_leaf` checks if the entry is a hugepage (always false on x86)
  - `pgd_none` returns true if the entry value is 0
  - `pgd_none_or_clear_bad` returns true if the entry value is 0 or invalid
  - `pgd_val` returns the entry value
  - `pgd_present` returns true if `_PAGE_PRESENT` is set
  - `pgd_populate` sets the entry value to point to a p4d table
  - `pgd_addr_end` rounds va up to the boundary or the end
    - `((va >> 48) + 1) << 48` or `end`, whichever is lower
  - `pgd_page_vaddr` returns the p4d table pointed to by the entry
- P4D / PML4 is similar
  - `p4d_t` represents a p4d entry
  - `p4d_alloc` allocates a p4d table for a va on demand
    - if `pgd_none` (p4d table for the va is missing), alloc the p4d table and
      `pgd_populate` to update pgd
    - return `p4d_offset`
  - `p4d_offset` returns the p4d entry for the va
    - `&p4d[(va >> 39) & 0xff]`, although `p4d` must be looked up from `mm->pgd`
  - `p4d_pgtable` returns the pud table pointed to by the entry
- PUD / PDP is similar
  - `pud_t` represents a pud entry
  - `pud_leaf` checks if the pud entry is a hugepage (`_PAGE_PSE` bit)
- PMD / PD
  - `pmd_t` represents a pmd entry
  - `pmd_alloc` allocates a pmd table for a va on demand
    - unlike upper tables, this inits a per-pmd-table spinlock rather than
      sharing `mm->page_table_lock`
  - `pmd_populate` takes a `struct page *` rather than a `pte_t *`
  - `pmd_install` locks per-pmd lock before calling `pmd_populate`
  - `pmd_pgtable` and `pmd_page_vaddr` return the pte table pointed to by the
    entry
- PTE / PT
  - `pte_t` represents a pte entry
  - `pte_alloc` allocates a pte table on demand
    - similar to pmd, this inits a per-pte-table spinlock
    - no va is specified
    - returns an err code intead of `pte_t *`
  - no `pte_offset`
  - `pte_offset_map` returns the pte entry for a va
  - no `pte_populate`
  - `set_pte` sets the entry value to point to a page
  - `set_ptes` sets contiguous entries to point to contiguous pages

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
