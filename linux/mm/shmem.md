Kernel shmem
============

## initialization and configs

- call sequence
  - `start_kernel`
  - `vfs_caches_init`
  - `mnt_init`
  - `shmem_init`
- `shmem_init` always registers a `file_system_type` named `tmpfs`, but the
  implementation depends on configs
  - it is always on because kernel uses this filesystem to manage shared
    memory
  - when `CONFIG_SHMEM` is not set, this filesystem calls into
    `ramfs_init_fs_context` and is effectively the same as `ramfs`
  - when `CONFIG_SHMEM` is set, this filesystem calls into
    `shmem_init_fs_context` and is the (full or partial) `tmpfs` we know
    - when `CONFIG_TMPFS` is not set, only enough fs features are implemented
      to support managing shared memory
    - only when `CONFIG_TMPFS` is set, a full-fledged fs is implemented

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
