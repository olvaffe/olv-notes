Kernel memory
=============

## Overview

- page fault handling
  - `handle_mm_fault` handles userspace faults
    - it is mainly called from arch fault irq
    - it walks the page table and allocates them as needed
    - at the last level, `handle_pte_fault`
      - if the pte entry is missing,
        - `do_fault` faults in the page from backing using `vma->vm_ops->fault`
        - `do_anonymous_page` allocs a new page for the anonymous mapping
      - if the pte entry misses the present bit (swapped out),
        - `do_swap_page` pages in the page from swap
- page table management
  - `unmap_page_range` zaps a va range
    - `clear_full_ptes` clears the pte entries
    - `__tlb_remove_folio_pages` frees the leaf pages
  - `free_pgtables` frees intermdeiate pgtables for a va range
    - it depends on `unmap_page_range`
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
    - the entry value encodes the pa of the page table of the next level, or
      the page in case of `pte_t`
  - each page table consists of an array of entries
  - a pointer to an entry (e.g., `pud_t *`) more commonly refer to a single
    entry
    - but sometimes, it points to the first entry of a table and refers to the
      entire table
- address translation
  - given a va and a pgd
  - the va is broken down into 5 indices and an offset
  - each index identifies an entry in a page table and each entry identifies
    the page table of th next level or the page
    - `pgd[i]` identifies the p4d table
    - `p4d[j]` identifies the pud table
    - `pud[k]` identifies the pmd table
    - `pmd[l]` identifies the pte table
    - `pte[m]` identifies the page
  - pa is thus `page + offset`
- a typical 64-bit cpu
  - all tables are `PAGE_SIZE` and are `PAGE_SIZE`-aligned
  - all entry types are 64-bit
    - a table is thus an array of `PAGE_SIZE / 8` entries
  - all entry types share a similar format
    - the lower N bits are the pa of the lower level page table or the page
      - N is often 52, but may also be 46, 56, etc.
    - because page table and page are `PAGE_SIZE`-aligned, the bottom bits are
      repurposed for flags
    - the highest `64 - N` bits are also flags
  - when `PAGE_SIZE` is 4KB,
    - each table is an array of 512 entries and requires 9-bit to index
    - a va thus consists of `9 * 5 + 12 = 57` bits
- page table allocation
  - `pgd_alloc` allocates a pgd table and returns the pointer to the table
    - there is a 1:1 relation between `mm_struct` and pgd tables
  - `p4d_alloc` allocates a p4d table on demand and returns the pointer to the
    p4d entry
    - it takes a pgd entry and a va
      - the pgd entry is from `pgd_offset(mm, va)`
    - if missing, `__p4d_alloc` allocs a p4d table and updates the pgd entry to
      point to the p4d table
    - `p4d_offset(pgd, va)` returns the pointer to the p4d entry
  - `pud_alloc` allocates a pud table on demand and returns the pointer to the
    pud entry
    - it takes a p4d entry and a va
    - if missing, `__pud_alloc` allocs a pud table and updates the p4d entry to
      point to the pud table
    - `pud_offset(p4d, va)` returns the pointer to the pud entry
  - `pmd_alloc` allocates a pmd table on demand and returns the pointer to the
    pmd entry
    - it takes a pud entry and a va
    - if missing, `__pmd_alloc` allocs a pmd table and updates the pud entry
      to point to the pmd table
      - note that the pmd table has its own lock instead of sharing
        `mm->page_table_lock`
    - `pmd_offset(pud, va)` returns the pointer to the pmd entry
  - `pte_alloc` allocates a pte table on demand and returns an error code
    - it takes a pmd entry but no va
    - if missing, `__pte_alloc` allocs a pte table and updates the pmd entry
      to point to the pte table
      - note that the pte table has its own lock instead of sharing
        `mm->page_table_lock`
- page table walk
  - given a mm and a va, we can walk the tables to find the pte entry
    - `pgd_offset(mm, va)` returns the pgd entry containing the va
    - `p4d_offset(pgd, va)` returns the p4d entry containing the va
    - `pud_offset(p4d, va)` returns the pud entry containing the va
    - `pmd_offset(pud, va)` returns the pmd entry containing the va
    - `pte_offset_map(pmd, va)` returns the pte entry containing the va
  - legacy `CONFIG_HIGHPTE`
    - there was a time when pte tables might be in highmem
    - that's why `pte_offset_map` has a `_map` suffix, implying
      `kmap_atomic()`
  - `pte_offset_map_lock` is a convenient function
    - `pte_lockptr` returns the spinlock for the pte table
      - `pmd_page(pmd)` reads the pmd entry value to get the pa of the pte
        table, and returns the page holding the pte table
      - `page_ptdesc` casts `page` to `ptdesc`
        - they have compatible physical layout
      - `ptlock_ptr` returns `ptdesc->ptl`, the page table lock
    - the convenient function returns the pte entry and the page table lock,
      with the lock locked
- page table zap
  - given a mm and a va (of a page),
    - we can do page table talk to find the pte entry
    - `vm_normal_page` returns the page pointed to by the pte entry
    - `should_zap_folio` returns false for anon folio unless `even_cows`
    - `clear_full_ptes` clears the pte entry
    - `folio_remove_rmap_ptes` decrements the page's map count in rmap
  - note how it only updates the pte entry
    - it does not free any page table thus does not update higher-level page
      table entries
  - this allows the pages holding the data to be reclaimed
    - `munmap` does this via `unmap_vmas`
    - `ftruncate` does this via `unmap_mapping_range`
    - `MADV_DONTNEED` does this via `zap_page_range_single_batched`
- page table free
  - given a mm and a page-aligned va range,
    - we can do page table talk to find any entry of any level touched by the
      va range
    - for each pte table touched by the va range,
      - `pmd_clear` clears the pmd entry
      - `pte_free_tlb` frees the pte table
      - this depends on zapping
        - we must zap all pte entries such that the pages pointed to can be
          reclaimed
        - this then free the pte table and update the pmd entry
    - for each pmd table fully covered by the va range,
      - `pud_clear` clears the pud entry
      - `pmd_free_tlb` frees the pmd table
    - for each pud table fully covered by the va range,
      - `p4d_clear` clears the p4d entry
      - `pud_free_tlb` frees the pud table
    - for each p4d table fully covered by the va range,
      - `pgd_clear` clears the pgd entry
      - `p4d_free_tlb` frees the p4d table
    - for each p4d table fully covered by the va range,
      - `pgd_clear` clears the pgd entry
      - `p4d_free_tlb` frees the p4d table
  - this matters most for pte tables because higher-level page tables are
    scarce
  - pgd is not freed here
  - this frees the pages holding the (mostly pte) page tables
    - `munmap` does this via `free_pgtables`
- when a process dies,
  - `exit_mmap` calls `unmap_vmas` to zap and calls `free_pgtables` to free
  - `mm_free_pgd` frees the pgd table

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
  - `vmf->pte` points to the pte entry and `vmf->orig_pte` is the entry value
    - `vmf->pte` is NULL when the pte table is not allocated yet
    - `vmf->pte` is also reset to NULL when the page is not allocated yet
      (`pte_none(vmf->orig_pte)`)
  - if no `vmf->pte`, `do_anonymous_page` faults in a new folio
  - if there is `vmf->pte` but not `pte_present`, `do_swap_page` swaps in
    - this happens when the folio is swapped out
- `do_anonymous_page` faults in a new page
  - `pte_alloc` allocs the pte table on demand
    - the pmd entry is updated to point to the pte table
  - `vmf_anon_prepare` allocs `vma->anon_vma` on demand for rmap
    - for a file-backed vma, rmap uses `vma->vm_file->f_mapping->i_mmap` to
      track all vmas of the file
    - for an anon vma, rmap allocs `vma->anon_vma` for the same purpose
  - `alloc_anon_folio` allocs a folio
  - `pte_offset_map_lock` returns the pte entry
  - `folio_add_new_anon_rmap` adds the folio to rmap
    - it sets `PG_swapbacked`
    - `__folio_set_anon` sets `folio->mapping` to `vma->anon_map`
      - `folio_test_anon` will return true
  - `folio_add_lru_vma` calls `folio_batch_move_lru` with `lru_add`
    - the real add is usually deferred until the batch is drained
    - `lru_add` adds the folio to `lruvec`
    - it sets `PG_lru`
  - `set_ptes` updates the pte entry to point to the folio
- if `shrink_folio_list` decides to reclaim an anon folio,
  - `folio_alloc_swap` calls down to `__swap_cache_add_folio`
    - it sets `PG_swapcache`
      - from this point on, even though the folio has no file backing and thus
        is not managed by any page cache, `folio_mapping` returns "swap
        cache", a special page cache which uses swap as the backing
      - `__remove_mapping` below will clear the bit shortly
    - `folio->swap` points to the swp entry
  - `try_to_unmap` calls `try_to_unmap_one` on each vma where the folio is
    mapped
    - `get_and_clear_ptes` clears the pte entry
    - because this is `folio_test_anon`, `swp_entry_to_pte` and `set_pte_at`
      encode the swp entry in the pte entry
      - the key is to ensure `_PAGE_PRESENT` (bit 0) is cleared
      - upon fault, `do_swap_page` will swap in from the encoded swp entry
  - if dirty, `pageout` writes back to the swap backing
  - `__remove_mapping` deletes the folio from the page cache
    - `__swap_cache_del_folio` deletes the folio from "swap cache" and clears
      `PG_swapcache`
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
- when `munmap` calls down to `unmap_region` to unmap an anonymous mapping,
  - `unmap_vmas` zaps pte entries
    - it walks the page tables
    - `zap_present_ptes` zaps a present pte entry
      - `vm_normal_page` returns the page pointed to by the pte entry
      - `should_zap_folio` returns true because `details->even_cows` is set
      - `folio_remove_rmap_ptes` removes the folio from rmap
      - `__tlb_remove_folio_pages` frees the folio
  - `free_pgtables` frees page tables that are no longer needed
  - `tlb_finish_mmu` performs the frees
    - all frees are deferred until this point after tls is flushed
    - `tlb_table_flush` frees page tables
      - `__tlb_remove_table_free` calls `free_page`
    - `tlb_batch_pages_flush` frees pages
      - `free_pages_and_swap_cache` calls `folios_put_refs`
        - `__page_cache_release` removes a folio from lru
        - `free_unref_folios` frees pages
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
