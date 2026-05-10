# Kernel mmap

## Overview

- `sys_mmap -> ksys_mmap_pgoff -> vm_mmap_pgoff -> do_mmap`
  - `vm_mmap_pgoff` locks the mm for the duration of `do_mmap`
  - `prot` is `PROT_*` from userspace
  - `flags` is `MAP_*` from userspace
  - `vm_flags` is initially 0
- `vm_flags` is updated
  - `calc_vm_prot_bits` converts all `PROT_*` to `VM_*`
    - `arch_calc_vm_prot_bits` converts arch-specific `PROT_*`
  - `calc_vm_flag_bits` converts some `MAP_*` to `VM_*`
    - `arch_calc_vm_flag_bits` converts arch-specific `MAP_*`
  - `mm->def_flags` is typically 0 unless `mlockall(MCL_FUTURE)`
  - `VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC`
- `__get_unmapped_area` finds an available starting addr for the mapping
  - `file->f_op->get_unmapped_area`
  - `shmem_get_unmapped_area`
  - `thp_get_unmapped_area_vmflags`
  - `mm_get_unmapped_area_vmflags`
- more checks and `vm_flags` may be updated again
- `mmap_region` ensures a vma is created
  - more checks
    - `arch_validate_flags` performs arch-specific checks
  - `__mmap_setup` preps
  - `call_mmap_prepare` uses the new `file->f_op->mmap_prepare` if supported
    - it gives the file a chance to adjust `vm_area_desc`
    - it updates `mmap_state` from the adjusted `vm_area_desc`
  - `__mmap_new_vma` allocs a new vma if needed
    - `vm_area_alloc` allocs
    - `vma` is initialized based on `mmap_state`
    - `__mmap_new_file_vma` calls `file->f_op->mmap` if the new
      `file->f_op->mmap_prepare` is not supported
      - `can_mmap_file` makes sure they are mutually exclusive
      - it gives the file a chance to adjust `vm_area_struct`
  - `__mmap_complete`

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
