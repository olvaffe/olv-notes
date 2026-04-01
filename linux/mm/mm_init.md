Linux MM Init
=============

## PID 0: `start_kernel`

- x86 `setup_arch`
  - `e820__memory_setup`
    - `e820__memory_setup_default` typically has 3 or 4 ram entries
      - `[0, ~640KB]`
      - `[1MB, ~2GB]`
      - `[4GB, ~REMAINING_RAM]`
  - `trim_bios_range` updates e820 to reserve first 4KB, etc.
  - `max_pfn` is set to `e820__end_of_ram_pfn`, which is last page
  - `max_low_pfn` is set to `e820__end_of_low_ram_pfn`, which is last page before 4GB
    - will be updated later
  - `e820__memblock_setup` calls `memblock_add` for each e820 ram entry
  - `init_mem_mapping` sets up page table for kernel linear map
    - it also updates `max_low_pfn` to `max_pfn`
  - `initmem_init` calls `x86_numa_init` to init numa
    - without `CONFIG_NUMA`, all memblock ranges belong to node 0
    - `numa_init`
      - `dummy_numa_init` fakes node 0 for all physical ram
      - `alloc_node_data` allocs `pg_data_t` and inits `NODE_DATA(nid)`
  - `x86_init.paging.pagetable_init` is `paging_init`
- arm `setup_arch -> early_init_dt_scan -> early_init_dt_scan_nodes`
  - `early_init_dt_scan_memory` parses nodes with `device_type = "memory"`
    - `early_init_dt_add_memory_arch` adds a memblock
  - `early_init_dt_check_for_usable_mem_range` parses extra ranges
- `mm_core_init_early` calls `free_area_init`
  - `arch_zone_limits_init` gets zone limits
    - `ZONE_DMA` is capped by `MAX_DMA_PFN` (16MB)
    - `ZONE_DMA32` is capped by `MAX_DMA32_PFN` (4GB)
    - `ZONE_NORMAL` is capped by `max_pfn`
  - `sparse_init` inits `mem_section` and `vmemmap`
    - `vmemmap` is the `struct page` array
  - zones are initialized according to zone limits
  - `for_each_mem_pfn_range` loops over memblock ranges
  - `for_each_node` loops over each node (only node 0)
    - `free_area_init_node` fully inits `NODE_DATA(nid)`, which is `pg_data_t`
    - `node_set_state` adds the nid to `N_MEMORY` and `N_NORMAL_MEMORY`
  - `calc_nr_kernel_pages` inits `nr_kernel_pages`
    - `for_each_free_mem_range` loops over memblock ranges that are not reserved
  - `memmap_init` inits memmap (`struct page` array)
    - `__init_single_page` inits a single `struct page`
    - `init_unavailable_range` sets `PG_reserved` on holes
      - `sparse_init` allocates `struct page` array in sections
      - each section covers 128MB
      - if a memblock range is not aligned, there are `struct page`s that do
        not correspond to physical ram
  - `set_high_memory` inits `high_memory` to point to end of ram
- `mm_core_init`
  - `build_all_zonelists` builds zone lists
    - `check_highest_zone` updates `policy_zone` to `ZONE_NORMAL`
  - `report_meminit` reports if pages are zeroed on alloc/free
    - `mem auto-init: stack:all(zero), heap alloc:on, heap free:off`
  - `memblock_free_all` frees pages from memblock to buddy

## PID 1: `kernel_init`

- `init_mm_internals` inits vmstats
- `page_alloc_init_late`
  - `mem_init_print_info` summaries mem info
- initcalls
- `free_initmem` frees kernel sections needed only for init
- exec to userspace

## NUMA Nodes

- there is a `struct pglist_data *node_data[MAX_NUMNODES];` array
  - one `pglist_data` for each node
- a `pglist_data` has many fields
  - `node_zones` are zones in this node
  - `__lruvec` is lruvec in this node
  - `vm_stat`
    - there are `NR_VM_NODE_STAT_ITEMS` items

## Zones

- a `zone` has many fields
  - `vm_stat`
    - there are `NR_VM_ZONE_STAT_ITEMS` items

## LRU

- a `lruvec` has many fields
  - `lists`
    - there are `NR_LRU_LISTS` lists

## Page Flags

- `PG_locked` means the page is locked
  - it functions as a per-page mutex
- `PG_writeback` means the page is being written back
  - it is set between `folio_start_writeback` and `folio_end_writeback`
- `PG_referenced` means the page was recently used
  - it is set on read, write, mmap, etc.
  - see `PG_active`
- `PG_uptodate` means the page holds the truth
  - it is set when the contents are synchronized with or newer than the
    backing
- `PG_dirty` means the page needs writeback
  - it is set when the contents are newer than the backing
- `PG_lru` means the page is on one of the lru lists
- `PG_head` means the page is the first of a compound page (hugepage)
- `PG_waiters` means `folio_waitqueue` of the page is non-empty
- `PG_active` means the page is on the active lru list
  - when reclaim sees
    - `+PG_active +PG_reference`: clear `PG_reference`
    - `+PG_active -PG_reference`: clear `PG_active` and move to inactive list
    - `-PG_active +PG_reference`: set `PG_active`, clear `PG_reference`, and
      move to active list
    - `-PG_active -PG_reference`: reclaim the page
- `PG_workingset` means the page is in the "working set"
  - e.g., it is set if it is re-faulted shortly after reclaim
- `PG_reserved` means the page is special
  - it tells mm to stay away from managing it: no reclaim, etc.
  - when an api expects a page, and to pass mmio addr to the api, kernel can
    fake a `PK_reserved` page?
- `PG_private` means `folio->private` has private fs data
  - when set, reclaim must ask fs to clear the bit before reclaim
- `PG_reclaim` means the page is being reclaimed
  - it is set when the reclaim requires writeback and takes time
  - it prevents another concurrnet reclaim
- `PG_swapbacked` means the page has no backing
  - it is set on allocation and never changes
- `PG_unevictable` means the page is on unevictable lru list
- `PG_dropbehind` means the page was used for streaming io and hit rate is low
  - e.g., `posix_fadvise(POSIX_FADV_NOREUSE)`
- `PG_mlocked` means the page is `mlock`ed
  - it causes `PG_unevictable` after the page is moved to the unevictable lru
    list
- `PG_readahead` means the page was paged in by readahead
  - if a page with the bit is accessed, it encourages readahead to read more
- `PG_swapcache` means the page is temporarily managed by swapcache
  - a private anonymous mapping has no backing and thus no page cache
    - that is, a process heap, stack, etc. has no `inode` backing and thus no
      `address_space`
  - to be able to page anonymous mapping out to swap device, its pages
    temporarily use swapcache as the page cache, which enable them to be paged
    out
  - as such, it is set when the page is managed by swapcache

## Page Refcounts

- `alloc_page` and `__free_page`
  - `alloc_page` allocs a page with refcount 1
    - `alloc_frozen_pages_noprof` allocs a page with refcount 0
    - `set_page_refcounted` sets refcount to 1
  - `__free_page` frees a page with refcount 1
    - `put_page_testzero` decrements refcount to 0
    - `__free_frozen_pages` frees a page with refcount 0
- `vm_operations_struct::fault`
  - `filemap_fault`, the generic implementation
    - `filemap_get_folio` returns a folio with incremented refcount
    - `vmf->page` is the folio
  - `do_fault`, the user
    - `__do_fault` calls the callback to get `vmf->page`
    - `finish_fault` assigns the folio to the pte entry, which is the owner
    - on errors, `folio_put` decrements the refcount
- `mm_struct::pgd`
  - the pgd table, all intermediate levels of tables, and the leaf pages form
    a tree
  - all pages are owned by the tree
  - `exit_mmap` frees all but the pgd table
    - `unmap_vmas` frees leaf pages and clears pte tables
    - `free_pgtables` frees all but the pgd table
  - `mm_free_pgd` frees the pgd table
- `filemap_add_folio` and `filemap_remove_folio`
  - `filemap_add_folio` calls `__filemap_add_folio`
    - `folio_ref_add` adds a ref
  - `filemap_remove_folio` calls `filemap_free_folio`
    - `folio_put_refs` removes a ref
- `folio_add_file_rmap_ptes` and `folio_remove_rmap_ptes`
  - they don't change refcount
- `folio_add_lru`
  - it does not change refcount
  - when `folio_put` drops the last ref, `page_cache_release` calls
    `lruvec_del_folio` automatically (but not atomically; the refcount is
    decremented first)
