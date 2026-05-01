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
- comparing to file-based mapping,
  - no `folio_alloc_swap` because a folio is always in page cache
  - `try_to_unmap` is the same
  - does not trigger extra writeback for dirty folio, but rely on regular
    writeback
  - `__remove_mapping` and `free_unref_folios` are the same
- `folio_alloc_swap`
- `swap_writeout`
- `__swap_cache_del_folio`

## `do_swap_page`

- swapping in happens in `do_swap_page`
  - `swap_cache_get_folio` tries to get the folio from the swapcache
  - if readback is required, `swap_readpage` or `swapin_readahead` perform the
    readback
- `swap_writepage` and `swap_readpage` submit bios
- when `handle_mm_fault` handles a page fault, if the pte has never been set
  up, `do_pte_missing` allocs the page and points pte to the page
- if the pte exists, but `pte_present` returns false, `do_swap_page` swaps the
  page in
- `pte_to_swp_entry` converts `pte_t` to `swp_entry_t`
- `swap_cache_get_folio` looks up the page from swapcache
- on swapcache miss, `swapin_readahead` swaps in the page with RA
  - this is considered a major page fault

## Userspace

- `swapon` syscall adds a swap device/file
  - `init_swap_address_space` creates an array of `address_space` for the swap
    device/file
    - every address space is 64MB (`SWAP_ADDRESS_SPACE_PAGES`)
    - the swap device/file is backed by this array of `address_space`s
  - the array of `address_space` is the swapcache
    - they are similar to a normal `address_space` for a normal flie, except the
      backing store is the swap file/device
