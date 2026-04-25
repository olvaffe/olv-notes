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

## `do_swap_page`

- when `handle_mm_fault` handles a page fault, if the pte has never been set
  up, `do_pte_missing` allocs the page and points pte to the page
- if the pte exists, but `pte_present` returns false, `do_swap_page` swaps the
  page in
- `pte_to_swp_entry` converts `pte_t` to `swp_entry_t`
- `swap_cache_get_folio` looks up the page from swapcache
- on swapcache miss, `swapin_readahead` swaps in the page with RA
  - this is considered a major page fault

## swap

- `swapon` syscall adds a swap device/file
  - `init_swap_address_space` creates an array of `address_space` for the swap
    device/file
    - every address space is 64MB (`SWAP_ADDRESS_SPACE_PAGES`)
    - the swap device/file is backed by this array of `address_space`s
  - the array of `address_space` is the swapcache
    - they are similar to a normal `address_space` for a normal flie, except the
      backing store is the swap file/device
- swapping out appears to happen in `shrink_folio_list`
  - pages on the anon lru list can be swapped out
    - pages on the file lru list can be written to the fs rather than swapped
      out
  - `add_to_swap` moves a folio to the swapcache
    - the folio is no longer owned by the pagecache of the anon file but by
      the swapcache
  - `pageout` calls `writepage` to write a dirty folio back to the backing
    storage
  - `folio_free_swap` removes the folio from the swapcache
- swapping in happens in `do_swap_page`
  - `swap_cache_get_folio` tries to get the folio from the swapcache
  - if readback is required, `swap_readpage` or `swapin_readahead` perform the
    readback
- `swap_writepage` and `swap_readpage` submit bios
