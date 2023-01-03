Kernel shmem
============

## shmem

- tmpfs is a pseudo-filesystem where files live on top of file page cache and
  swap cache
  - only when `CONFIG_TMPFS` is set, tmpfs gains many essential features to
    meet userspace expectation
  - on the other hand, there is always an internal mount known as shmem that
    does not rely on `CONFIG_TMPFS` features
- `shmem_file_setup` creates a file in shmem
  - it creates a new inode in the mount
  - it then createa a file for the new inode with `shmem_file_operations`
- `shmem_aops` can migrate a page to swap
- `shmem_file_operations`
  - `get_unmapped_area` finds an available address range for mmap
  - `mmap` sets `vma->vm_ops` to `shmem_vm_ops`
- `shmem_vm_ops` handles page faults
  - `map_pages` is a generic one that maps a couple more pages around in the
    hope that those pages will get used soon
  - `fault` calls `shmem_getpage_gfp`.  It finds a page from cache, from swap,
    or it allocates.  The page is added to both the page cache and the swap
    lru
- `shmem_read_mapping_page` also calls `shmem_getpage_gfp`
