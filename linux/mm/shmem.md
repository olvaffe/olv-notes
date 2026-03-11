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

- this sections list changes enabled by `CONFIG_TMPFS`
- `CONFIG_TMPFS` enables user mounts, and relevant features, for `tmpfs`
  - it implies `CONFIG_SHMEM`
- `struct file_system_type shmem_fs_type`
  - `shmem_init_fs_context` sets `SB_I_VERSION` for nfs
  - `shmem_fs_parameters` provides mount options for user mounts
- `struct fs_context_operations shmem_fs_context_ops`
  - `shmem_get_tree` calls `shmem_fill_super`
    - it applies default options for user mounts
    - it supports `shmem_export_ops`
    - it sets `SB_NOSEC` to bypass `file_remove_privs` as an optimization
  - `shmem_parse_monolithic` parses legacy mount options as a string
  - `shmem_parse_one` parses modern mount options individually
  - `shmem_reconfigure` is for remount
- `struct export_operations shmem_export_ops` is for nfs
- `struct super_operations shmem_ops`
  - `shmem_statfs` is for `df`, etc.
  - `shmem_show_options` shows options in `/proc/mounts`
- `struct inode_operations shmem_dir_inode_operations`
  - all dir ops are for user mounts
  - for kernel mounts, we need the dir inode for `/` but don't need dir ops
- `struct inode_operations shmem_short_symlink_operations` is for symlinks
- `struct inode_operations shmem_symlink_inode_operations` is also for
  symlinks, where filename is stored in swappable page
- `struct file_operations shmem_file_operations`
  - all fops except open/mmap are for user mounts
- `struct address_space_operations shmem_aops`
  - `shmem_write_begin` is used by `shmem_file_write_iter` indirectly
  - `shmem_write_end` is used by `shmem_file_write_iter` indirectly

## inode

- for root, `shmem_fill_super` calls `shmem_get_inode` with
  - `idmap` is `nop_mnt_idmap`
  - `sb` is per-shmem-mount
  - `dir` is NULL
  - `mode` is `S_IFDIR | 0777 | S_ISVTX`
    - user mounts can override with `-O mode`
  - `dev` is 0
  - `flags` is `VMA_NORESERVE_BIT`
- for kernel mounts, `__shmem_file_setup` calls `shmem_get_inode` with
  - `idmap` is `nop_mnt_idmap`
  - `sb` is per-shmem-mount
  - `dir` is NULL
  - `mode` is `S_IFREG | S_IRWXUGO`
  - `dev` is 0
  - `flags` is from the caller
- for user mounts, `shmem_mknod` calls `shmem_get_inode` with
  - `idmap` is from vfs
  - `sb` is per-shmem-mount
  - `dir` is the parent dir
  - `mode` is from vfs
  - `dev` is from vfs
  - `flags` is `VMA_NORESERVE_BIT`
- `shmem_get_inode`
  - `shmem_reserve_inode` returns a unique ino
  - `new_inode` calls `shmem_alloc_inode` to alloc a `shmem_inode_info`
  - it sets ops depend on `mode & S_IFMT`
    - dir sets `inode->i_op` to `shmem_dir_inode_operations`
    - file sets
      - `inode->i_op` to `shmem_inode_operations`
      - `inode->i_fop` to `shmem_file_operations`
      - `inode->i_mapping->a_ops` to `shmem_aops`
    - special sets `inode->i_op` to `shmem_special_inode_operations`

## In-Kernel Users

- `shmem_file_setup` creates and opens an inode
  - `shmem_acct_size` accounts for the size, unless `VMA_NORESERVE_BIT`
  - `shmem_get_inode` creates the inode
  - `inode->i_size` is set to `size`
  - `alloc_file_pseudo` "opens" the inode
    - this simply marks the file `FMODE_OPENED`
    - with `vfs_open`, `do_dentry_open` calls `f_op->open` before marking
      `FMODE_OPENED`
- `shmem_read_folio_gfp` populates the page cache at the specified page index
  - `filemap_get_entry` looks up the folio at the index
  - `xa_is_value` returns true if the folio is swapped out
    - during swap out, `shmem_writeout` calls `shmem_delete_from_page_cache`
      to replace the folio by swp entry, a "value"
    - `shmem_swapin_folio` swaps it back in
  - if there is already a folio, todo
  - else, `shmem_alloc_and_add_folio` allocs a folio and populates the page
    cache

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
