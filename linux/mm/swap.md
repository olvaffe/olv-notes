# Kernel swap

## Overview

- `swap.c` is always enabled
  - it provides helpers to add folios to lru lists or move folios between lru
    lists
  - it applies to both anonymous mapping and file-based mapping, and is not
    specific to swap
  - `folio_add_lru` adds a folio to lru
  - `folio_mark_accessed` promotes a folio to the next state
    - from low to high, there are 4 states
      - inactive list
      - inactive list, referenced/accessed
      - active list
      - active list, referenced/accessed
    - each reclaim scan demotes a folio to the prev state
- `swap_state.c` provides a special page cache, "swap cache", for anonymous
  mappings
  - file-based mappings get their pages from the files' page caches
    - during reclaim, lru sync pages to the backing, deletes them from page
      caches, and frees them
  - anonymous mappings get their pages from buddy directly
    - during reclaim, lru makes "swap cache" their temporary page caches. lru
      then sync pages to swap, delete them from "swap cache", and frees them
  - `__swap_cache_add_folio` and `swap_cache_del_folio` add/remove a folio
    to/from "swap cache"
  - `swapin_readahead` populates swap cache
- `swapfile.c` provides special block devices as the backing for "swap cache"
  - `swapon` and `swapoff` add/remove special block devices
  - `folio_alloc_swap` and `folio_free_swap` alloc/free a swap entry for a
    folio
    - they call `__swap_cache_add_folio` and `swap_cache_del_folio`
      automatically to update "swap cache"
- `page_io.c` provides io helpers to access swap block devices
  - `swap_writeout` writes folio data to swap
  - `swap_read_folio` reads swap data to folio

## Swap Out

- `shrink_folio_list` can swap out an anon folio in an anon mapping
  - if the anon folio is not in sway cache yet, `folio_alloc_swap` adds the
    folio to the swap cache
    - the folio is obviously dirty when this happens
  - if the anon folio is still mapped in any pgtable, `try_to_unmap` unmaps it
    from pgtables
    - the folio refcount typically drop to 2, 1 from swap cache and 1 from the
      folio list
  - if the anon folio is dirty, `swap_writeout` issues a write io
    - because the folio is dirty before io completion, it is not reclaimed
      immediately
  - if all good, `__remove_mapping` removes the folio from the swap cache and
    `free_unref_folios` frees the folio to buddy
    - this happens later in another call, when the folio is no longer dirty
- in the case of file-based mapping,
  - no `folio_alloc_swap` because a folio is always in page cache
  - `try_to_unmap` is the same
  - does not trigger extra writeback for dirty folio, but rely on regular
    writeback
  - `__remove_mapping` and `free_unref_folios` are the same
- `folio_alloc_swap` adds a folio to the swap cache
  - `swap_alloc_fast` calls `alloc_swap_scan_cluster` to allcoate an swp entry
    from `percpu_swap_cluster`
  - `swap_alloc_slow` locks `swap_avail_lock` and calls
    `cluster_alloc_swap_entry` to allocate an swp entry from `swap_avail_head`
    - `alloc_swap_scan_list` calls `alloc_swap_scan_cluster` as well
  - `alloc_swap_scan_cluster` loops through all possible ranges
    - `cluster_scan_range` checks if a range is available
    - `__swap_cluster_alloc_entries` calls `__swap_cache_add_folio` to add the
      folio
      - `__swap_table_set` marks the swp entry used
      - `folio_ref_add` increments refcount now that the folio is managed by
        the swap cache
      - `folio_set_swapcache` marks the `PG_swapcache`
      - `folio->swap` is set to the swp entry
      - `NR_FILE_PAGES` and `NR_SWAPCACHE` stats are increments
- `try_to_unmap` calls `try_to_unmap_one` to unmap the folio from pgtables
  - `get_and_clear_ptes` clears the pte entry to 0
  - `folio_dup_swap` increments the refcount of the swp entry
    - this is the number of pte entries who values are replaced by swp entries
  - `swp_entry_to_pte` encodes swp entry as a pte entry val
    - specifically, the present bit is cleared and `pte_present` is false
  - `set_pte_at` sets the pte entry to the val
  - now that the folio is unmapped, accessing the foilo results in page fault
    - before swap out, it is minor fault because the folio is still in swap cache
    - after swap out, it is major fault
- `swap_writeout` writes a dirty folio to storage asynchronously
  - `folio_free_swap` returns true if the dirty folio does not need writeout
    - `folio_maybe_swapped` checks if the swp entry refcount is non-zero
      - see `folio_dup_swap` above: refcount is the number of pte entries that
        reference the swp entry
      - refcount is 0 when, for example, the process has freed the folio
    - instead of writeout, `swap_cache_del_folio` removes the folio from the
      swap cache
  - `arch_prepare_to_swap` can be used to save the arm mte tag
  - if `is_folio_zero_filled`, `swap_zeromap_folio_set` marks so in swp entry
    than writeout
  - `__swap_writepage` calls `swap_writepage_bdev_async` to submit write bio
    - it does not wait for completion
- `__swap_cache_del_folio` removes the folio from swap cache
  - `__remove_mapping` has called `folio_ref_freeze` to set folio refcount to 0

## Swap In

- after swap out,
  - the swp entry refers to the storage where the folio contents are saved
  - all pte entries for the folio have been replaced by encoded swp entry
    - `pte_present` returns false for the encoded swp entry
  - the swp entry has a refcount that is the number of replaced pte entries
- `do_swap_page` is called by `handle_pte_fault` to handle a page fault
  - `softleaf_from_pte` converts `pte_t` to `swp_entry_t`
  - `swap_cache_get_folio` looks up the page from swap cache
  - `swapin_readahead` swaps in a folio with RA, if the original folio has
    been dropped from swap cache
    - this is considered a major page fault
  - `folio_lock_or_retry` locks the folio
    - if there is in-flight read bio to update the folio, this acquires the
      lock after the read io completes
  - `arch_swap_restore` can be used to restore the arm mte tag
  - `folio_ref_add` increments the refcount
  - `set_ptes` updates pte entry
- `swapin_readahead` typically call `swap_vma_readahead` to allocate a folio
  and emit a read bio
  - `swap_vma_ra_win` determines the number of folios to readahead
  - `swap_cache_alloc_folio` allocates a folio
    - `swap_cache_get_folio` returns the folio from the swap cache, if exists
    - `folio_alloc_mpol` allocates a new folio
    - `__swap_cache_prepare_and_add` adds the new folio to swap cache
  - `swap_read_folio` calls `swap_read_folio_bdev_async` to submit read bio
    - it does not wait for completion

## Userspace

- `swapon` syscall adds a swap device/file
  - `init_swap_address_space` creates an array of `address_space` for the swap
    device/file
    - every address space is 64MB (`SWAP_ADDRESS_SPACE_PAGES`)
    - the swap device/file is backed by this array of `address_space`s
  - the array of `address_space` is the swapcache
    - they are similar to a normal `address_space` for a normal flie, except the
      backing store is the swap file/device
