Kernel filemap
==============

## page cache

- where is the page cache?
  - every `struct inode` has `i_mapping` that points to `i_data`
  - `i_data` is a `struct address_space`, which is the page cache for the
    inode
  - `address_space::i_pages` is an array of `struct folio *`
    - the type is `xarray`, which is a sparse array of pointers

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
  - `filemap_read_folio` reads the contents from the backing storage

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
