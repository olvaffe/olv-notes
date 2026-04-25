# Kernel shmem

## Overview

- `shmem_init` always registers `tmpfs` and does a kernel mount for in-kernel
  users
  - `start_kernel -> vfs_caches_init -> mnt_init -> shmem_init`
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

## `shmem_get_inode`

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

## `shmem_get_folio_gfp`

- `shmem_get_folio_gfp` is mostly called from
  - `shmem_fault` with `SGP_CACHE` and `vmf`
  - `shmem_read_folio_gfp` with `write_end` and `SGP_CACHE`
  - if user mount,
    - `shmem_write_begin` with `write_end` and `SGP_WRITE`
    - `shmem_file_read_iter` with `SGP_READ`
    - `shmem_fallocate` with `write_end` and `SGP_FALLOC`
- `filemap_get_entry` looks up the folio at the file index in the page cache
  - if cache hit, it returns the folio
  - if cache miss, it returns NULL
  - if swapped out, it returns "swp entry"
    - `folio` is not a valid pointer but encoded swp entry
    - `xa_is_value` returns true
- if swapped out, `shmem_swapin_folio` takes care of the rest
- if cache hit, we want to know if the folio is up-to-date
  - if `folio_test_uptodate`, done
  - otherwise, the folio was merely fallocated and still contains garbage
  - but if `SGP_READ`, we can return NULL and let `shmem_file_read_iter` read
    from the zero page
- `shmem_alloc_and_add_folio` allocs a folio
  - `shmem_alloc_folio` allocs a folio
  - `PG_swapbacked` is set
  - `shmem_add_to_page_cache` adds the folio to page cache
  - `folio_add_lru` adds the folio to lru
- if not `folio_test_uptodate`, clear the folio to zero and `folio_mark_uptodate`
  - if `SGP_WRITE`, this is skipped as an optimization because caller might
    overwrite the entire page

## Swap

- when `shrink_folio_list` reclaims a shmem folio, `pageout` calls
  `shmem_writeout` to swap it out
  - if not `folio_test_uptodate`, the page was merely fallocated and contains
    garbage
    - zero it now and `folio_mark_uptodate`
  - `folio_alloc_swap` adds the folio to swap cache
    - `__swap_cache_add_folio` inits `folio->swap` and sets `PG_swapcache`
  - `shmem_swaplist` tracks all swapped inodes for `shmem_unuse`
    - `swapoff` calls `shmem_unuse` before removing a swap dev
  - `folio_dup_swap` increments the swp entry refcount
    - from 0 to 1 if this is a new swp entry
  - `shmem_delete_from_page_cache` deletes the folio from the page cache
    - it replaces the folio by `swp_to_radix_entry` instead of NULL
    - `xa_is_value` will return true and the value is encoded swp entry
  - `swap_writeout` submits io to the swap device
    - `AOP_WRITEPAGE_ACTIVATE` means errors and tells the caller to
      re-activate the folio (move it to the active list)
- when `shmem_get_folio_gfp` gets a shmem folio, if it is swapped out,
  `shmem_swapin_folio` swaps it in
  - `swap_cache_get_folio` looks up the folio in swap cache
    - this happens only in the process of reclaim, after `shmem_writeout` adds
      it to swap cache and before `__remove_mapping` removes it
    - this is minor fault because no io is needed
  - `shmem_swap_alloc_folio` or `shmem_swapin_cluster` reads in the data
    - this becomes `VM_FAULT_MAJOR` due to io
  - `folio_wait_writeback` waits for reclaim writeback
    - this may happen with minor fault
  - if the folio is in the wrong zone, `shmem_replace_folio` allocs a new
    folio and copies the contents
  - `shmem_add_to_page_cache` adds the folio to the page cache
  - `folio_put_swap` decrements the swp entry refcount
  - `swap_cache_del_folio` removes the folio from the swap cache
  - `folio_mark_dirty` dirties the folio
    - because its backing in swap device is gone

## `CONFIG_TRANSPARENT_HUGEPAGE`

- this sections list changes enabled by `CONFIG_TRANSPARENT_HUGEPAGE`
- cmdline options
  - `__setup("thp_shmem=", setup_thp_shmem)`
    - `huge_shmem_orders_*` defaults to 0
  - `__setup("transparent_hugepage_tmpfs=", setup_transparent_hugepage_tmpfs)`
    - `tmpfs_huge` config-time defauls to `SHMEM_HUGE_NEVER`
  - `__setup("transparent_hugepage_shmem=", setup_transparent_hugepage_shmem)`
    - `shmem_huge` config-time defaults to `SHMEM_HUGE_NEVER`
- sysfs attrs
  - `__ATTR_RW(shmem_enabled)`
    - `shmem_huge` and `SHMEM_SB(shm_mnt->mnt_sb)->huge` are updated
  - `__ATTR(shmem_enabled, 0644, thpsize_shmem_enabled_show, thpsize_shmem_enabled_store)`
    - `huge_shmem_orders_*` are updated
- `shmem_init`
  - override `SHMEM_SB(shm_mnt->mnt_sb)->huge` to `shmem_huge`
  - `huge_shmem_orders_inherit` defaults to `BIT(HPAGE_PMD_ORDER)` (2MB)
- `struct fs_context_operations shmem_fs_context_ops`
  - `shmem_get_tree` calls `shmem_fill_super`
    - `sbinfo->huge` defauls to `tmpfs_huge`
      - this is the only place `tmpfs_huge` is used
      - `shmem_init` will override `sbinfo->huge` to `shmem_huge` for
        `shm_mnt`
- `struct super_operations shmem_ops`
  - `shmem_unused_huge_count` is to reclaim shmem-specific internal caches
  - `shmem_unused_huge_scan` is to reclaim shmem-specific internal caches
- `struct file_operations shmem_file_operations`
  - `shmem_get_unmapped_area` find an unused va range for the mapping. If
    huge, it ensures that the range is huge-aligned.
- `shmem_get_inode` calls `mapping_set_large_folios` if `sbinfo->huge`
  - it informs vfs that shmem ops can handle hugepages
  - e.g., when `mapping_large_folio_support`, `page_cache_ra_order` allocs
    hugepages for readahead
- `shmem_get_folio_gfp`
  - `shmem_allowable_huge_orders` returns allowed orders
  - `shmem_alloc_and_add_folio` potentially allocs a huge folio
  - if the hugepage is larger than the inode size, add the inode to
    `sbinfo->shrinklist`
    - during reclaim, it can be splitted such that the unused pages can be
      reclaimed
- `shmem_swapin_folio`
  - if swap readback fails to read back a hugepage, `shmem_split_large_entry`
    splits a huge swp entry stored in the page cache into multiple regular swp
    entries
- `shmem_writeout`
  - if inode size smaller than the hugepage, split before swap out

## In-Kernel Users

- `shmem_file_setup` creates and opens an inode
  - `shmem_acct_size` accounts for the size, unless `VMA_NORESERVE_BIT`
  - `shmem_get_inode` creates the inode
  - `inode->i_size` is set to `size`
  - `alloc_file_pseudo` "opens" the inode
    - this simply marks the file `FMODE_OPENED`
    - with `vfs_open`, `do_dentry_open` calls `f_op->open` before marking
      `FMODE_OPENED`
- `shmem_read_folio_gfp` is a wrapper for `shmem_get_folio_gfp`

## User Mounts

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
