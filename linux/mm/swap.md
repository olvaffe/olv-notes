Kernel swap
===========

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
