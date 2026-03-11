Kernel shmem
============

## Overview

- `shmem_init` always registers `tmpfs` and does a kernel mount for in-kernel
  users
  - with `CONFIG_TMPFS` (which implies `CONFIG_SHMEM`), `tmpfs` supports
    swapping, user mounts, mount options, etc.
  - with only `CONFIG_SHMEM`, `tmpfs` supports swapping but no user mounts
    (`SB_NOUSER`)
  - without `CONFIG_SHMEM`, `tmpfs` is the same as `ramfs`. It does not
    support swapping but it supports user mounts.
- some in-kernel users
  - `rootfs` and `devtmpfs` are the same as `tmpfs` or `ramfs`, depending on
    `CONFIG_TMPFS`
  - shared anonymous mappings map shmem files
  - shared `/dev/zero` mappings map shmem files
  - memfd fds are shmem files
  - drm makes heavy use of shmem

## `CONFIG_TMPFS`

- `CONFIG_TMPFS` enables user mounts, and relevant features, for `tmpfs`
  - it implies `CONFIG_SHMEM`
- `struct file_system_type shmem_fs_type`
  - `shmem_init_fs_context` sets `SB_I_VERSION` for nfs
  - `shmem_fs_parameters` provides mount options for user mounts
- `struct fs_context_operations shmem_fs_context_ops`
  - `shmem_parse_monolithic` parses legacy mount options as a string
  - `shmem_parse_one` parses modern mount options individually
  - `shmem_reconfigure` is for remount
- `struct super_operations shmem_ops`
  - `shmem_statfs` is used by `df`, etc.
  - `shmem_show_options` shows options in `/proc/mounts`
- `struct inode_operations shmem_dir_inode_operations`
  - all dir ops are for user mounts
  - for kernel mounts, we need the dir inode for `/` but don't need dir ops
- `struct file_operations shmem_file_operations`
  - all fops except open/mmap are for user mounts
- `struct address_space_operations shmem_aops`
  - `shmem_write_begin` is used by `shmem_file_write_iter` indirectly
  - `shmem_write_end` is used by `shmem_file_write_iter` indirectly
- `struct vm_operations_struct shmem_vm_ops`
- `struct vm_operations_struct shmem_anon_vm_ops`

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

## tmpfs

- `shmem_init` calls `register_filesystem` to register tmpfs
- when userspace mounts an instance,
  - `shmem_init_fs_context` inits
    - `fc->fs_private` to `shmem_options`
    - `fc->ops` to `shmem_fs_context_ops`
  - `shmem_get_tree` inits
    - `sb->s_fs_info` to `shmem_sb_info`
      - this is tmpfs's private sb info for this instance
    - `sb->s_op` to `shmem_ops`
    - `sb->s_root` to `d_make_root(inode)`
      - `shmem_get_inode` allocs an inode
- when userspace creates a file,
  - `shmem_create`
    - `shmem_get_inode` allocs an inode
    - `d_make_persistent` inits `dentry->d_inode` and more
- when userspace opens a file,
  - `shmem_file_open` does very little
- when userspace reads a file,
  - `shmem_file_read_iter`
    - `shmem_get_folio` returns the folio at the offset
    - `copy_folio_to_iter` copies the data from the folio to the buf
- when userspace mmaps a file,
  - `shmem_mmap_prepare` inits `desc->vm_ops` to `shmem_anon_vm_ops`
- when userspace faults,
  - `shmem_fault` faults in a folio
