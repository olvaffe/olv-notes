# Kernel mmap

## `mmap`

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
    - `map->prev` and `map->next` point to neighboring vmas
    - `map->mas_detach` points to conflicting vmas
      - this can happen with `MAP_FIXED`
    - `may_expand_vm` checks against `RLIMIT_AS` and `RLIMIT_DATA`
    - `accountable_mapping` returns true for writable private mapping
    - `vms_clean_up_area` unmaps conflicting vmas
      - this can happen with `MAP_FIXED`
    - `set_desc_from_map` init `vm_area_desc` from `mmap_state`
  - `call_mmap_prepare` uses the new `file->f_op->mmap_prepare` if supported
    - it gives the file a chance to adjust `vm_area_desc`
    - it updates `mmap_state` from the adjusted `vm_area_desc`
  - `vma_merge_new_range` tries to merge with neighboring vmas
  - if cannot merge, `__mmap_new_vma` allocs a new vma
    - `vm_area_alloc` allocs a vma
    - `vma` is initialized based on `mmap_state`
    - `vma` is further initialized depending on `map->file`
      - if `map->file` and the new `file->f_op->mmap_prepare` is not
        supported, `__mmap_new_file_vma` calls `file->f_op->mmap`
        - `can_mmap_file` has made sure they are mutually exclusive
        - it gives the file a chance to adjust `vm_area_struct`
      - else if share anon, `shmem_zero_setup` creates a shmem for the vma
      - else if priv anon, `vma_set_anonymous` sets `vma->vm_ops` to NULL
    - `vma_iter_store_new` adds the vma to mm
    - `vma_link_file` adds the vma to `vma->vm_file->f_mapping->i_mmap`
      - this is for rmap, where `mapping->i_mmap` tracks all vmas
      - for priv anon, `vmf_anon_prepare` will create `vma->anon_vma` on
        demand for the same purpose
  - `__mmap_complete`
    - `vms_complete_munmap_vmas` removes conflicting vmas
    - `vma_set_page_prot` updates `vma->vm_page_prot` based on `vma->vm_flags`

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
