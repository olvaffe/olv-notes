Kernel stat
===========

## `SYSCALL_DEFINE2(stat)`

- it calls `vfs_statx`
- `vfs_getattr` gets the attrs from the inode
  - it calls fs-specific `inode->i_op->getattr` which also calls
    `generic_fillattr`

## `notify_change`

- `notify_change` is called from syscalls that change inode attrs
  - `chmod`, `chown`, `utimensat`, `truncate`, etc.
  - it calls fs-specific `inode->i_op->setattr`
- `ATTR_SIZE` requires special handling
  - if the new size is smaller, extra data are discarded
  - if the new size is larger, extra data are initialized to 0
    - most fs supports sparse and does not allocate blocks
    - that is, `st_blocks` does not increase
