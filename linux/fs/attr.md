# Kernel attr

## `getattr`

- `vfs_getattr_nosec` is called from
  - `SYSCALL_DEFINE2(stat, ...)`
  - `SYSCALL_DEFINE5(statx, ...)`
  - etc.
- `vfs_getattr_nosec` calls `inode->i_op->getattr` or the generic
  `generic_fillattr`
  - `generic_fillattr` copies stats from `inode` to `kstat`
  - fs `getattr` callback typically calls `generic_fillattr` as well

## `setattr`

- `notify_change` is called from
  - `SYSCALL_DEFINE2(utime, ...)`
  - `SYSCALL_DEFINE2(truncate, ...)`
    - no `SYSCALL_DEFINE4(fallocate, ...)`
  - `SYSCALL_DEFINE2(chmod, ...)`
  - `SYSCALL_DEFINE3(chown, ...)`
  - etc.
- `notify_change` calls `inode->i_op->setattr` or the generic `simple_setattr`
  - `simple_setattr`
    - `truncate_setsize` handles `ATTR_SIZE`
    - `setattr_copy` copies the attr to inode
    - `mark_inode_dirty` marks the inode dirty
  - fs `setattr` callback typically calls `setattr_copy` and
    `mark_inode_dirty` directly
- eventually, writeback calls `writeback_sb_inodes`
  - `do_writepages` calls `a_ops->writepages` to write dirty pages
  - `write_inode` calls `s_op->write_inode` to write dirty inode
- `ATTR_SIZE` requires special handling
  - if the new size is smaller, extra data are discarded
  - if the new size is larger, extra data are initialized to 0
    - most fs supports sparse and does not allocate blocks
    - that is, `st_blocks` does not increase
