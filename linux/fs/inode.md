# Linux VFS inode

## Overview

- an `inode` is an in-memory representation of an fs's on-disk inode
  - on path lookup, `filename_lookup -> path_lookupat -> link_path_walk`
    - it walks the path components and caches dentries/inodes along the path
    - `walk_component -> lookup_slow -> __lookup_slow`
      - `d_alloc_parallel` allocs a dentry
      - `inode->i_op->lookup` parses the on-disk inode into the dentry
        - `new_inode` allocs an inode
        - it looks up and parses the on-disk inode under the parent dir
        - `d_splice_alias` associates the inode with the dentry
        - it returns non-NULL when there is an existing dentry, informing the
          caller to destroy the duplicated dentry
  - a `dentry` associates a path component with an inode, cached for faster lookup
    - `dentry->d_parent` is the parent dir
    - `dentry->d_name` is the component name
    - it also supports negative cache, for when a component does not exist
    - it is managed by dcache
  - a `path` represents a full path
    - it consits of both dentry and mount
    - imagine `/boot/vmlinuz` of esp
    - the dentry is `vmlinuz` and the parent is esp `/`
    - it needs the mount to know the fs is mounted at `/boot`
  - on reclaim, `super_cache_scan` scans `sb->s_dentry_lru` and
    `sb->s_inode_lru` for unreferenced dentries and inodes
- `inode` interesting fields
  - `i_sb` and `i_ino` uniquely identify the inode
  - `i_uid`, `i_gid`, `i_size`, `i_atime`, `i_mtime`, `i_ctime`, etc. are stats
  - `i_mode` has both permissions and types
    - `S_IFSOCK`, socket
    - `S_IFLNK`, symbolic link
      - a symbolic link is an inode that refers to another inode
      - all hard links are the same inode with different dentries
    - `S_IFREG`, regular file
    - `S_IFBLK`, block device
    - `S_IFDIR`, directory
    - `S_IFCHR`, character device
    - `S_IFIFO`, FIFO
  - `i_op` is `struct inode_operations`
    - filesystems use different `inode_operations` for inodes of different types
    - it can manipulate stats, etc.
    - many ops are only interesting for `S_IFDIR`, to
      lookup/create/remove/rename child inodes
  - `i_fop` is `struct file_operations` for default fops
  - `i_mapping` is `struct address_space`
    - it is in-memory reprentation of the inode's data, aka page cache
    - it gives a linear in-memory view of the scattered on-disk data
    - there is `struct address_space_operations`
