Kernel memory
=============

## Page Faulting

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

## Page Tables

- there are 5 levels of page tables
  - `pgd_t`, `p4d_t`, `pud_t`, `pmd_t`, and `pte_t`
  - each type is arch-defined and represents an entry in a page table of the
    level
  - it is common that, on a 64-bit cpu,
    - all types are 64-bit
    - all tables are `PAGE_SIZE`
    - IOW, at each level, a table takes up a page and has `PAGE_SIZE / 8`
      entries
- `pgd_alloc` allocates a pgd table
  - it returns `pgd_t *`, which is a pointer to an array of `pgd_t`
  - while arch-specific, it is commont to allocate with
    - `(pgd_t *)get_zeroed_page(GFP_PGTABLE_USER)`
    - that is, a pgd table takes up a page
  - also arch-specific, but a `pgd_t` commonly contains the address of a p4d
    table
    - a p4d table also takes up a page
    - the address is thus page-aligned and the lower `PAGE_SHIFT` bits are
      flags of the entry
- `p4d_alloc` allocates a p4d table
  - the goal is to allocate a p4d table for a va, and update a pgd entry to
    point to the p4d table
  - `pgd_offset(mm, va)` can be used to find the pgd entry for the va
  - arch-specific `p4d_alloc_one` allocates the table
    - again, `(p4d_t *)get_zeroed_page` suffices
- `pud_alloc`, `pmd_alloc`, and `pte_alloc` work similarly
  - just remember they each allocates a table (an array of entries) of the
    corresponding level
- conversely, given a mm and a va, we can walk the tables to find the pte
  entry for the va
  - this is `follow_pte`
  - `pgd_offset(mm, va)` returns the pgd entry containing the va
  - `p4d_offset(pgd, va)` returns the p4d entry containing the va
  - `pud_offset(p4d, va)` returns the pud entry containing the va
  - `pmd_offset(pud, va)` returns the pmd entry containing the va
  - `pte_offset_map(pmd, va)` returns the pte entry containing the va
    - we need many pte tables and there was a time when we put pte tables in
      highmem (`CONFIG_HIGHPTE`)
    - that's why this function has a `_map` suffix, implying `kmap_atomic()`
  - `pte_offset_map_lock` is a convenient function
    - `pte_lockptr` returns `page->ptl`, where `page` is the page holding the
      pte table
      - `ptl` stands for page table lock
    - the convenient function returns the pte entry and the page table lock,
      with the lock locked
