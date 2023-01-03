Kernel filemap
==============

## page cache

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
- where is the page cache?
  - every `struct inode` has `i_mapping` that points to `i_data`
  - `i_data` is a `struct address_space`
  - `address_space::i_pages` is an array of pages and is the page cache for
    the inode
