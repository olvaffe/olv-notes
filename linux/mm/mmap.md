Kernel mmap
===========

## Memory Management

- `struct mm_struct`: the address space of a task
- `struct vm_area_struct` describes a continuous area, with same protection, of the address space
  - It has `struct vm_operations_struct`
  - It has `struct file *` for backing store
- `struct vm_struct`: a continuous kernel virtual area
- `struct address_space` can be viewed as the MMU of an a on-disk object.
  Accessing to the pages of the object goes through its `struct address_space`
  in its inode.

## Adding a VMA

- `vm_area_alloc` allocates a `vm_area_struct`
- the caller sets up `vm_area_struct` manually, such as
  - `vm_start` and `vm_end` for the addresses
  - `vm_page_prot` and `vm_flags` for prot and flags
  - `vm_ops` for accessing the vma
    - according to `vma_is_anonymous`, it is non-null unless the vma is anon
  - more
- `insert_vm_struct` inserts the vma into mm
  - `find_vma_intersection` makes sure the addr range is available
  - `vma_link` adds the vma to the mm
    - mm manages its vma using `mm->mm_mt`, a maple tree
    - if the vma is file-backed, `__vma_link_file` adds the vma to the
      `address_space` of the file
      - `address_space` is the page cache backing the file
      - `address_space` manages its vmas using
        `vma->vm_file->f_mapping->i_mmap`

## userspace mmap

- Call chain
  - `mmap`: userspace mmap call
  - `sys_mmap`: kernel space mmap call
  - `ksys_mmap_pgoff`: generic mmap function
  - `vm_mmap_pgoff`
  - `do_mmap_pgoff`
  - `get_unmapped_area`: to find an available starting addr
    (`addr & ~PAGE_MASK` means error!)
    - `arch_get_unmapped_area`
  - `mmap_region`: a new vma is created (a common case) and `file->f_op->mmap`
    is called

## Memory Mapping

- map pages in...
  - kernel space, physically contiguous: kmap, kmalloc
    - `kmap` is `page_address`; `kunmap` is no-op
      - `page_address` is `page_offset_base + ((page - vmemmap_base) << 12)`
    - `kmalloc` allocates physically contiguous pages and kmap them
  - device space: `dma_map_sg`
    - in the simplest case, `phys_to_dma(page_to_phys(virt_to_page(data)))`
    - takes only some shifts and some alus
  - user sapce: `remap_pfn_range`
    - update a user `vm_area_struct->mm` to map its addresses to
      different pages
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

## `copy_from_user`

- `__get_user_asm` copies 1, 2, or 4 from userspace
  - It directly `mov src,dst`.
  - If no fault, done.
  - If fault, it jumps to `do_page_fault`.
  - If the fault comes from an invalid address, we will reach `no_context`
    - this is where the fixup happens
    - The snippet `_ASM_EXTABLE(1b, 3b)` means if a fault happens at 1b, jumps
      to 3b to fix it.
    - at 3b, an error code is set and it jumps to 2b, where normal execution is
      resumed.
  - If the fault comes from an swapped out area, it calls `handle_mm_fault`.
- If the user pointer points to an mmaped shmem, it probably goes
  `handle_pte_fault`, `do_linear_fault`, and `__do_fault`.
  - `vma->vm_ops->fault` is called to fault in the page.
  - The new page is returned through `vmf->page`.
