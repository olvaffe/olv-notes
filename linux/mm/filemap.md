Kernel filemap
==============

## page cache

- where is the page cache?
  - every `struct inode` has `i_mapping` that points to `i_data`
  - `i_data` is a `struct address_space`, which is the page cache for the
    inode
  - `address_space::i_pages` is an array of `struct folio *`
    - the type is `xarray`, which is a sparse array of pointers
  - `address_space::a_ops` is the ops provided by the fs
    - e.g., fs translates a file read to bio reads according to fs on-disk
      format
  - page cache sits between file operations and bios, such that each inode
    appears as an array of folios
    - direct io bypasses page cache
- folio manipulations
  - `folio_lock` locks a folio
  - `folio_unlock` unlocks a folio
- page cache manipulations
  - `filemap_add_folio` adds a folio to the page cache (`address_space`)
  - `filemap_remove_folio` removes a folio from the page cache
  - `filemap_get_entry` returns the folio at the specified index
  - `filemap_get_folio` returns the folio at the specified index in fancy
    ways, such as alloc-on-demand, wait-for-writeback, etc.
- folio readback/writeback
  - `filemap_read_folio` reads data to bring a folio uptodate
  - `filemap_fdatawrite` starts writeback with `WB_SYNC_ALL`
  - `filemap_flush` starts writeback with `WB_SYNC_NONE`
  - `filemap_fdatawait` waits for writeback
  - `filemap_write_and_wait_range` starts and waits for writeback
- file operations
  - `generic_file_read_iter` is for read fop
    - it populates page cache and reads from the page cache
  - `generic_file_write_iter` is for write fop
    - it populates page cache and writes to the page cache
  - `generic_file_mmap_prepare` or (legacy) `generic_file_mmap` is for mmap fop
    - they use `generic_file_vm_ops` for vm ops, which obviously fault in
      folios from page cache

## folio

- compount pages
  - a `page` used to refer a single page
  - compount pages were introduced and a `page` may be compound and refer to
    contiguous single pages
  - for functions that accept a `struct page *`, some can handle compound
    pages and some can't
- `struct folio`
  - a `struct folio *` is essentially a `struct page *`
  - for functions that accept a `struct folio *`, they promise to support both
    single pages and compound pages
- `filemap_get_folio` returns the `struct folio *` at the given index
  - `index` is the `index`-th page of the mapping
  - it is also the index into the `address_space::i_pages` array
  - it can return NULL if the page is not cached yet
- `read_mapping_page` return the `struct page *` at the given index
  - unlike `filemap_get_folio`, if the page is not cached yet, it will
    allocate the page and initialize it from the backing storage
  - `filemap_alloc_folio` allocates a 0-order `folio` (i.e., a single `page`)
  - `filemap_add_folio` adds the folio to the xarray
    - and lru with `folio_add_lru`
  - `filemap_read_folio` reads the contents from the backing storage
- when a page that is from page cache, not from `alloc_page`, the page should
  be freed with `folio_put`, `put_page`, or `pagevec_release`

## reads

- `struct file_operations` has `read_iter` and `write_iter` that are used when
  a file descriptor is `read` or `write`
  - for example, ext4 uses `ext4_file_read_iter` and `ext4_file_write_iter`
- `struct address_space_operations` has `read_folio` and
  `write_begin`/`write_end` that are used to interact with the page cache
  - for example, ext4 uses `ext4_read_folio`, `ext4_write_begin`, and
    `ext4_write_end`
- when a file descriptor is read,
  - `read_iter` op often calls down to `filemap_read`
  - `filemap_get_pages` gets pages from page cache
    - if the contents are in the page cache, it can return right away
    - otherwise, `filemap_create_folio` uses `read_folio` op to populate the
      page cache and get the pages
    - it also uses readahead to populate the page cache
