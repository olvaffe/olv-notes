Kernel swap
===========

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
