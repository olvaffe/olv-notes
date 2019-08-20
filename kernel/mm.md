Kernel Memory Management
========================

## Kernel Address Space

* layout

    [PAGE_OFFSET ~ high_memory] ~ VMALLOC_OFFSET ~ [VMALLOC_START ~ VMALLOC_END] ~ 8KB ~ [PKMAP_BASE ~ FIXADDR_START ~ 4GB]
* `high_memory = min(physical memory, MAXMEM)`
* `MAXMEM = VMALLOC_END - PAGE_OFFSET - __VMALLOC_RESERVE (128MB)`
* persistent map is used by kmap
* fixed-mapped linear address is, for example, used by `kmap_atomic`

* the first area (encolosed by the first pair of brackets)
  * It is linear.
  * MMU is set up to be `addr -> (addr - PAGE_OFFSET)` in `init_memory_mapping`.
  * The first 4M of physical memory is mapped using 4k page entry.  The reset of
    lowmem is mapped using 4M page entry.
* the second area
  * For vmalloc, ioremap, etc.
  * pte is updated on demand through page fault handler, `do_page_fault`.
  * each subarea is described by `vm_struct`.
  * see also exception.txt
* the third area
  * used by kmap for high memory.
  * A small fixed adress space (`permanent_kmaps_init`), pointing to dynamically
    mapped pages.  The address space is fixed and is already in every process's
    page table. Changes to PTE propogate to all processes automatically.
* `__FIXADDR_TOP = 0xfffff000`


## Memory Management

* `struct mm_struct`: the address space of a task
* `struct vm_area_struct` describes a continuous area, with same protection, of the address space
  - It has `struct vm_operations_struct`
  - It has `struct file *` for backing store
* `struct vm_struct`: a continuous kernel virtual area
* `struct address_space` can be viewed as the MMU of an a on-disk object.
  Accessing to the pages of the object goes through its `struct address_space`
  in its inode.

## `copy_from_user`

* `__get_user_asm` copies 1, 2, or 4 from userspace
  * It directly `mov src,dst`.
  * If no fault, done.
  * If fault, it jumps to `do_page_fault`.
  * If the fault comes from an invalid address, we will reach `no_context`
    * this is where the fixup happens
    * The snippet `_ASM_EXTABLE(1b, 3b)` means if a fault happens at 1b, jumps
      to 3b to fix it.
    * at 3b, an error code is set and it jumps to 2b, where normal execution is
      resumed.
  * If the fault comes from an swapped out area, it calls `handle_mm_fault`.
* If the user pointer points to an mmaped shmem, it probably goes
  `handle_pte_fault`, `do_linear_fault`, and `__do_fault`.
  * `vma->vm_ops->fault` is called to fault in the page.
  * The new page is returned through `vmf->page`.

## userspace mmap

* Call chain
  * mmap: userspace mmap call
  * sys_mmap: kernel space mmap call
  * do_mmap_pgoff
  * get_unmapped_area: to find an available starting addr (`addr & ~PAGE_MASK` means error!)
    * `arch_get_unmapped_area`
  * mmap_region: a new vma is created (a common case) and `file->f_op->mmap` is called

## x86-64 Paging Table

* 4-level paging
  * 48-bit virtual address and 46-bit physical address
  * CR3 bit 51..12: PA to the 4KB-aligned PML4 (Page Map Level 4) table
    * 512 64-bit entries called PML4Es
  * VA bit 47..39: select one of the 512 PML4Es
    * each PML4E contains a PA to a PDP (page-directory-pointer) table
    * there are 512 64-bit entries called PDPEs
  * VA bit 38..30: select one of the 512 PDPEs
    * each PDPE contains a PA to a PD (page directory)
    * there are 512 64-bit entries called PDEs
  * VA bit 29..21: select one of the 512 PDEs
    * each PDE contains a PA to a page table
    * there are 512 64-bit entries called PTEs
  * VA bit 20..12: select one of the 512 PTEs
    * each PTE contains a PA to a page
  * VA bit 11..0: 12-bit offset into the page
* 5-level paging
  * 57-bit virtual address and 52-bit physical address
  * CR3 bit 51..12: PA to the 4KB-aligned PML5 (Page Map Level 5) table
  * VA bit 56..48: select one of the 512 PML5Es
    * each PML5E contains a PA to a PML4 table
    * there are 512 64-bit entries called PML5Es
* All entries at all levels have a similar (but different) format
  * bit 63..52: 12-bit for flags
    * bit 63 is NX (no execute)
    * more
  * bit 51..12: 40-bit pfn
  * bit 11..0: 12-bit for flags
    * bit 4 is page cache disabled (for the next table; or page when it is PTE)
    * bit 3 is page write through (for the next table; or page when it is PTE)
    * bit 1 is writable
    * bit 0 is present
    * more

## Nested Paging

* AMD-V Nested Paging White Paper
  * <http://developer.amd.com/wordpress/media/2012/10/NPT-WP-1%201-final-TM.pdf>
* SW shadow page table
  * guest has a guest page table
  * hypervisor has a shadow page table
  * the HW MMU uses the shadow page table when the guest is active
  * the guest page table is marked read-only
    * whenever the guest updates it, it traps into the hypersor
    * the hypervisor updates both the guest and the shadow page tables
* HW nested page table
  * HW MMU uses the guest page table directly
  * an additional nested page table set up by the hypervisor is used to
    translate guest physical address to host physical address
  * for a 4-level paging walk in guest results in 5 walks in the nested page
    walker (to access PML4, PDPE, PDE, PTE, and the page)
* memory type selection
  * MTRR
    * 0x0: UC
    * 0x1: WC
    * 0x4: WT
    * 0x5: WP (write protected)
    * 0x6: WB
    * Linux does not use MTRR.  BIOS(?) sets MTRRdefType to 0xc06, which means
      WB by default.
  * `IA32_PAT` MSR has 8 3-bit page attribute fields, PA0..PA7
    * each field can have one of these values
      * 0x0: UC
      * 0x1: WC
      * 0x4: WT
      * 0x5: WP
      * 0x6: WB
      * 0x7: UC-
    * In `pat_init`, they are initialized to
      * PA0: WB
      * PA1: WC
      * PA2: UC-
      * PA3: UC
      * PA4: WB (unused)
      * PA5: WP
      * PA6: UC- (unused)
      * PA7: WT
  * PTE's bit 7 (PAT), 4 (PCD), and 3 (PWT) form a 3-bit index that is used to
    select from PA0..PA7
    * `pgprot_writecombine` maps to PA1
  * In EPT (extended page table, used by KVM), MTRR is ignored and EPT bits
    5:3 replace MTRR (0: UC, 1: WC, 4: WT, 5 WP, 6: WB)
    * `vmx_get_mt_mask`

## x86-64

- Kconfig
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
- `start_kernel`
  - `setup_arch`
    - set up memblock according to e820, `e820__memblock_setup`
      - `memblock_add` adds a memory region to memblock
      - `memblock_reserve` adds a reserved region to memblock
        - a range that is in the memory regions but not in the reserved
          regions are considered available
      - an allocation finds a range in the memory regions but not in the
      	reserved regions, and then adds the range to the reserved regions
    - `init_mem_mapping` initializes `PAGE_OFFSET` mapping
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

## Memory Initialization (old)

* x86-64 uses 4-level page table.
  * `PGDIR_SHIFT == 39`
  * `PUD_SHIFT == 30`
  * `PMD_SHIFT == 21`.
  * `PTRS_PER_PGD` is 512, `1 << (48 - 39)`.
  * `PTRS_PER_PUD` is 512, `1 << (39 - 30)`.
  * `PTRS_PER_PMD` is 512, `1 << (30 - 21)`.
  * `PTRS_PER_PTE` is 512, `1 << (21 - 12)`.
* There are three memory allocators
  * brk
  * memblock
  * e820 and `alloc_low_page`
  * buddy allocator
* In `start_kernel`, after the linux banner is printed, `setup_arch` is called.
  * It calls `e820__memory_setup` to decide physical memory maps.
  * It calls `e820__end_of_ram_pfn` to decide 
    * `max_pfn`, (pfn of the last page of the DRAM) + 1.
    * `max_arch_pfn`, `1 << MAX_PHYSMEM_BITS`
  * It calls `e820__end_of_low_ram_pfn` to decide
    * `max_low_pfn`, which is `min(max_pfn, 4GB>>12)`
  * It calls `reserve_brk` to resert brk region.
  * It calls `init_mem_mapping` to map the low memory.
    * It runs before bootmem for early direct access.
    * The first range is the first 4MB (`PMD_SIZE`), using 4K page.
    * The second range is the rest, using 4M page
    * It calls `find_early_table_space` to reserve in `e820_table_start` and
      `e820_table_end` a place to store the page tables.
      * This storage is available to others through `alloc_low_page`.  It is
        the `MAPPING_BEYOND_END` mapped in `head_32.S` 
      * This storage is reserved by `reserve_early`.
    * The mappings are set up by `kernel_physical_mapping_init`.
    * cr3 is loaded, with `swapper_pg_dir` still being the PDE table.
    * `max_low_pfn_mapped` and `max_pfn_mapped` are set to `max_low_pfn`
      if everything goes normally.
    * That is, the mapped memory goes from 8MB to whole low memory.
  * It calls `initmem_init` to set up bootmem, boot-time physical memory
    allocator.
    * `e820_register_active_regions` is called with node 0 and all physical
      memory.  All e820 ram regions are stored in `early_node_map` of buddy
      allocator.
    * `setup_bootmem_allocator` is called.  It sets `after_bootmem` to 1.
    * It `reserve_early` an area for `bootmap`.
    * Later, `early_res_to_bootmem` is called to migrate early reserved area to
      bootmem map.
  * It calls `paging_init` to allocate an array of all `struct page *` from
    bootmem.
    * All pages are initialized as reserved at this point by `memmap_init_zone`.
* Much later in `start_kernel`, `mem_init` is called
  * It calls `free_all_bootmem` to return unused pages in bootmem to the buddy
    allocator.
  * `zap_low_mappings` is called to, forget low memory mappings.  That is,
    virtual address `< PAGE_OFFSET` is no longer mapped.  This is so that one
    can only access kernel memory from kernel address space.
* An example of early reserved regions

    (8 early reservations) ==> bootmem [0000000000 - 00377fe000]
      #0 [0000000000 - 0000001000]   BIOS data page ==> [0000000000 - 0000001000]
      #1 [0000001000 - 0000002000]    EX TRAMPOLINE ==> [0000001000 - 0000002000]
      #2 [0000006000 - 0000007000]       TRAMPOLINE ==> [0000006000 - 0000007000]
      #3 [0000100000 - 00004cb180]    TEXT DATA BSS ==> [0000100000 - 00004cb180]
      #4 [00004cc000 - 00004cf000]    INIT_PG_TABLE ==> [00004cc000 - 00004cf000]
      #5 [000009fc00 - 0000100000]    BIOS reserved ==> [000009fc00 - 0000100000]
      #6 [0000007000 - 0000008000]          PGTABLE ==> [0000007000 - 0000008000]
      #7 [0000008000 - 000000f000]          BOOTMAP ==> [0000008000 - 000000f000]
  * PGTABLE is one page because only the first 4M uses 4K page.  The rest uses
    4M page and does not need to allocate memory.

## brk section

* Introduced in `93dbda7cbcd70a0bd1a99f39f44a9ccde8ab9040`, 2009.
  * And modified several times after.
* In the linker script, `.brk` is reserved for `*(.brk_reservation)` and
  is delimited by `__brk_base` and `__brk_limit`.  It comes just after `.bss`
  and just before `.end`.
* `RESERVE_BRK` is used to reserve brk space
  * It creates `.brk_reservation` section with the given size
  * It is used to reserve space for init page tables and dmi
* When `head_32.S` creates `default_entry`, it creats page tables in `.brk`.
  * `_brk_end` marks the location after the page tables.
  * `extend_brk` is used to alloc space in brk.  It extends `_brk_end`.
    * It is used indirectly by `dmi_alloc` in `drivers/firmware/dmi_scan.c`.
* `reserve_brk` is called to reserve brk as `reserve_early`.  It also pins the
  brk as read-only.
  * It reserves up to `_brk_end`, not to `_brk_limit`.  Therefore, it is safe to
    `RESERVE_BRK` a large (safe) region.

## Memory Mapping

- map pages in...
  - kernel space, physically contiguous: kmap, kmalloc
    - `kmap` is `page_address`; `kunmap` is no-op
      - `page_address` is `page_offset_base + ((page - vmemmap_base) << 12)`
    - `kmalloc` allocates physically contiguous pages and kmap them
  - kernel space, virtually contiguous: vmap, vmalloc
    - vmap kmallocs a `vmap_area`, which carves out a region from
      `VMALLOC_START` 32TB address space, and inserts it into
      `vmap_area_root`.  It kmallocs a `vm_struct` to save the carved-out
      region together with the flags, and sets up the paging table to point to
      the pages
    - vmalloc is similar, except it also `alloc_page` pages
  - device space: `dma_map_sg`
    - in the simplest case, `phys_to_dma(page_to_phys(virt_to_page(data)))`
    - takes only some shifts and some alus
  - user sapce: `remap_pfn_range`
    - update a user `vm_area_struct->mm` to map its addresses to
      different pages
- `ioremap` is similar to `vmap` but for MMIO addresses
  - it is arch-specific
  - it sets cache modes for the mapping
  - it gets a `vm_struct` like in vmap, and calls the generic
    `ioremap_page_range` to update the page tables
- `vm_mmap` maps a file into userspace
  - it does many checks, and then allocates a new `vm_area_struct`
  - it saves `VM_xxx` flags to `vma->vm_flags`, saves arch-specific bits
    for page protection in `vma->vm_page_prot`, and saves the file, if any, to
    `vma->vm_file`
  - it calls the `mmap` op of the file with the vma.  The op is allowed to
    make many changes to the vma.
    - for anonymous mmap, there is no file.  For `VM_SHARED`,
      `shmem_zero_setup` is called to set up a shmem file.  For `VM_PRIVATE`,
      the VM is truely anonymous and has no `vma->vm_ops`
  - it adds the vma into `current->mm`
- pfn is `pa >> PAGE_SHIFT`
  - when pa is system memory, `pfn_to_page` can be used to get the `struct page`
  - when pa is MMIO, there is no `struct page`; `pfn_valid` returns false
  - `pfn_t` is pfn plus flags to distinguish the differences
    - `PFN_DEV`: no `struct page`
- VMA has many flags that the file `mmap` op can set
  - `VM_IO` means to treat the area as if it is backed by MMIO
  - `VM_PFNMAP` means to treat the area as if there is no `struct page`
  - `VM_MIXEDMAP` means to treat the area as if there may or may no be `struct page`
  - `VM_DONTEXPAND` means to disallow mremap expanding

## Page Faulting

- On x86, `X86_TRAP_PF` jumps to `page_fault` in `entry_64.S` which calls
  `do_page_fault`
  - for userspace fault, it calls `find_vma` to find the VMA and calls
    `handle_mm_fault`
- the mm code for fault starts in `handle_mm_fault`
- a `vm_fault` is set up; page table until pte is set up
- for true anonymous (`MAP_ANONYMOUS | MAP_PRIVATE`) vma, `do_anonymous_page`
  - prepare `vma->anon_vma`, which is not a vma at all btw
  - allocate a page from the buddy allocator
  - add the page to `anon_vma`
  - set up pte to point to the page
- normally, `do_fault`
  - the `vma->vm_ops->fault` op can return a page in `vmf->page`, or set up pte
    itself with `vmf_insert_*` and return `VM_FAULT_NOPAGE`
  - nothing further is needed when `VM_FAULT_NOPAGE` is returned
  - but when a page is returned, vmf owns a reference to the page.  Normally
    the return value is `VM_FAULT_LOCKED` indicating the page is also locked
    (in-use).  `do_fault` goes on to sets up the pte, which gives the
    reference to pte, and adds the page to rmap.  It unlocks the page before
    it exits.
  - rmap knows which ptes point to a given page
  - `MAP_ANONYMOUS | MAP_SHARED` creates a shmem file during mmap and follows
    this normal path
- COW follows the anonymous vma path a bit
  - it also prepares `vma->anon_vma` and allocates a page from the buddy
    allocator
  - it additionally `copy_user_highpage` from the original page to the COW
    page

## page, page cache, and swap

- pages are allocated with `alloc_page` or `alloc_pages`
  - they are freed with `__free_page` or `__free_pages`
- in fs, pages are allocated the same way, but they are also added to the page
  cache, `add_to_page_cache`, and to the swap lru cache, `lru_cache_add`
  - it turns out pages are refcounted: `page_ref_xxx`
    - page cache owns references to the managed pages
    - swap cache also owns references to the managed pages
  - when a driver has a page that is from fs, not from `alloc_page`, the page
    should be freed with `put_page` or `pagevec_release`
  - it also has a `_mapcount` which is the number vmas the page is mapped
    - when a pte is taken down in `zap_pte_range`, `page_remove_rmap` is called
      to reduce `_mapcount`.  The referernce to the page is transferred to tlb
      in `__tlb_remove_page`
- amazingly, it also embeds a 1-bit semaphore
  - `lock_page` indicates down
  - `unlock_page` indicates up
- when a page is allocated, it can specify the reclaim flags
  - `__GFP_DIRECT_RECLAIM` means the caller would rather wait than failing the
    allocation
  - `___GFP_KSWAPD_RECLAIM` means the allocation can fail, but please wake up
    kswapd
- kswapd thread calls `kswapd_shrink_node` to reclaim pages from various
  sources to the buddy allocator
- specifically, `shrink_page_list` is given a list of pages to reclaim
  - it locks the page with `lock_page`
  - it checks `mapping_unevictable`
  - it `try_to_unmap` the pages from all vmas
  - if the page is in a page cache, and have no other references, removing it
    from the page cache and taking away the last references in
    `__remove_mapping`
  - it does many other things; if all good, the pages are added to
    `free_pages`
  - then those pages are returned to the buddy allocator with
    `free_unref_page_list`

## shmem

- tmpfs is a pseudo-filesystem where files live on top of file page cache and
  swap cache
  - only when `CONFIG_TMPFS` is set, tmpfs gains many essential features to
    meet userspace expectation
  - on the other hand, there is always an internal mount known as shmem that
    does not rely on `CONFIG_TMPFS` features
- `shmem_file_setup` creates a file in shmem
  - it creates a new inode in the mount
  - it then createa a file for the new inode with `shmem_file_operations`
- `shmem_aops` can migrate a page to swap
- `shmem_file_operations`
  - `get_unmapped_area` finds an available address range for mmap
  - `mmap` sets `vma->vm_ops` to `shmem_vm_ops`
- `shmem_vm_ops` handles page faults
  - `map_pages` is a generic one that maps a couple more pages around in the
    hope that those pages will get used soon
  - `fault` calls `shmem_getpage_gfp`.  It finds a page from cache, from swap,
    or it allocates.  The page is added to both the page cache and the swap
    lru
- `shmem_read_mapping_page` also calls `shmem_getpage_gfp`
