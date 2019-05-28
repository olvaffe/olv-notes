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

## shmem

* It is `/dev/shm`, mounted as tmpfs?
* It uses five kinds of operations
  * `struct super_operations`
  * `struct address_space_operations`
  * `struct file_operations`
  * `struct inode_operations`
  * `struct vm_operations_struct`

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
* All entries at all levels have a standand format
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
