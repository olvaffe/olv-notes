Kernel memory
=============

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
