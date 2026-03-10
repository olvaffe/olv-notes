Kernel memory
=============

## Overview

- page fault handling
  - `handle_mm_fault` handles userspace faults
    - it is mainly called from arch fault irq
    - it walks the page table and allocates them as needed
    - at the last level, `handle_pte_fault`
      - if the pte is missing,
        - `do_fault` faults in the page from backing using `vma->vm_ops->fault`
        - `do_anonymous_page` allocs a new page for the anonymous mapping
      - if the pte is not present,
        - `do_swap_page` pages in the page from swap
- page table management
  - `unmap_single_vma` zaps (tears down) page table for a vma
    - `munmap` zaps and destroys vma
    - `zap_page_range_single` zaps vma but does not destroy it
  - `copy_page_range` copies page table for `fork`
- userspace memory mapping
  - `vm_insert_page` inserts a page to userspace page table, for mmap
  - `remap_pfn_range` inserts a pfn range to userspace page table, for mmap
    - it is named "remap" because it remaps pfns that are already accessible
      to the kernel to the userspace
  - `vmf_insert_pfn` inserts a pfn to userspace page table, for fault handling

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
- given a mm and a va, we can walk the tables to find the pte entry for the va
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

## Page Faulting

- the mm code for fault starts in `handle_mm_fault`
- a `vm_fault` is set up
- page table down until pmd is set up
  - `pgd_offset` returns the pgd entry for the va; `mm->pgd` is already allocated
  - `p4d_alloc` returns the p4d entry for the va; p4d table is allocated on demand
  - `pud_alloc` returns the pud entry for the va; pud table is allocated on demand
  - `pmd_alloc` returns the pmd entry for the va; pmd table is allocated on demand
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

## Anonymous Mapping

- when `mmap` calls down to `__mmap_new_vma` to create a new vma,
  - `vm_area_alloc` inits `vma->vm_ops` to `vma_dummy_vm_ops`
  - if there is a file, `file->f_op->mmap` can override
  - if `VM_SHARED`, `shmem_zero_setup` overrides to `shmem_anon_vm_ops`
  - else, `vma_set_anonymous` overrides to NULL
- `vma_is_anonymous` checks for `vma->vm_ops`
- when there is a fault, `handle_pte_fault` handles the fault
  - if no `vmf->pte`, `do_anonymous_page` faults in a new folio
  - if there is `vmf->pte` but not `pte_present`, `do_swap_page` swaps in
- `do_anonymous_page` faults in a new page
  - `pte_alloc` allocs the pte table on demand
    - the pmd entry is updated to point to the pte table
  - `vmf_anon_prepare` allocs `vma->anon_vma` on demand for rmap
    - for a file-backed vma, rmap uses `vma->vm_file->f_mapping->i_mmap` to
      track all vmas of the file
    - for an anon vma, rmap allocs `vma->anon_vma` for the same purpose
  - `alloc_anon_folio` allocs a folio
  - `pte_offset_map_lock` returns the pte entry
  - `folio_add_new_anon_rmap`
    - it sets `PG_swapbacked`
    - `__folio_set_anon` sets `folio->mapping` to `vma->anon_map`
  - `folio_add_lru_vma` calls `folio_batch_move_lru` with `lru_add`
    - the real add is usually deferred until the batch is drained
    - `lru_add` adds the folio to `lruvec`
    - it sets `PG_lru`
  - `set_ptes` updates the pte entry to point to the folio
- if `shrink_folio_list` decides to reclaim an anon folio,
  - `folio_alloc_swap` calls down to `__swap_cache_add_folio`
    - it sets `PG_swapcache`
      - from this point on, even though the folio has no file backing and thus
        is not managed by page cache, `folio_mapping` returns "swap cache", a
        special page cache which uses swap as the backing
    - `folio->swap` points to the swp entry
    - the folio is now managed by "swap cache"
  - `try_to_unmap` calls `try_to_unmap_one` on each vma where the folio is
    mapped
    - `get_and_clear_ptes` clears the pte entry
    - because this is `folio_test_anon`, `swp_entry_to_pte` and `set_pte_at`
      encode the swp entry in the pte entry
      - the key is to ensure `_PAGE_PRESENT` (bit 0) is cleared
  - if dirty, `pageout` writes back to the swap backing
  - `__remove_mapping` calls `__swap_cache_del_folio` to delete the folio from
    "swap cache"
    - it clears `PG_swapcache`
  - `free_unref_folios` frees the page back to buddy
- `do_swap_page` faults in a page by swapping
  - `softleaf_from_pte` decodes the pte entry back to softleaf entry of type
    `SOFTLEAF_SWAP`
  - if the folio has been swapped out from the "swap cache",
    - `swapin_readahead` swaps it in again
      - `swap_cache_alloc_folio` allocs the folio and adds it to lru
    - the return code is changed from 0 (success without io) to
      `VM_FAULT_MAJOR` (success with io)
  - `pte_offset_map_lock` returns the pte entry
  - `folio_add_anon_rmap_ptes` adds the folio to rmap
  - `set_ptes` updates the pte entry to point to the folio
- comparing to file-based mapping,
  - `do_anonymous_page` becomes `do_fault`
  - reclaim takes more or less the same path
    - no need for `folio_alloc_swap` because there is already a page cache
    - `try_to_unmap` does not encode swp entry in the pte entry
  - `do_swap_page` also becomes `do_fault`
- comparing to shmem-based mapping,
  - `do_anonymous_page` becomes `do_fault -> shmem_fault`
  - reclaim takes more or less the same path
    - no need for `folio_alloc_swap` because there is already a page cache
    - `try_to_unmap` does not encode swp entry in the pte entry
    - if dirty, `pageout` calls `shmem_writeout` instead of `swap_writeout`
      - `folio_alloc_swap` is called here to alloc swp entry
      - `swp_to_radix_entry` encodes swp entry as xa value
      - `shmem_delete_from_page_cache` replaces the folio by the encoded swp
        entry in `mapping->i_pages`
  - `do_swap_page` also becomes `do_fault -> shmem_fault`
    - in this case, because `xa_is_value` returns true, `shmem_swapin_folio`
      swaps in
