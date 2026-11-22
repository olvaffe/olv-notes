# Linux VFS Super Block

## Overview

- a `super_block` is an in-memory representation of an fs's on-disk super block
  - on mount, `do_mount -> path_mount -> do_new_mount -> do_new_mount_fc -> fc_mount`
    - `vfs_get_tree -> fc->ops->get_tree -> get_tree_bdev`
      - it creates `super_block` on demand
      - it calls `fill_super` to parse the on-disk super block, which is also
        expected to parse the on-disk root inode into `sb->s_root`
  - it is destroyed when the last mount goes away
  - for each fs, there is exactly one `super_block`
    - two mounts can share the same super block
- `super_block` interesting fields
  - `sb->s_op` points to fs's `struct super_operations`
    - it allows VFS to manipulate the specific fs
    - `alloc_inode` allocates and initializes`struct inode`
  - `sb->s_root` is the dentry to the root inode of the fs
