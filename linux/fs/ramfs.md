Kernel ramfs
============

## overview

- it is the simplest possible fs
- do not use ramfs
  - files always stay in pagecache and cannot be swapped out
  - fs size is unbounded
- use tmpfs instead

## what does a minimal fs look like?

- it calls `register_filesystem` to register its `file_system_type`
  - there is a `init_fs_context` callback
- when the `init_fs_context` callback is called, set its
  `fs_context_operations`
  - there is a `get_tree` callback
- when the `get_tree` callback is called, create a `super_block` and
  initialize the super block
  - this initializes `super_operations` and the root `inode`
- each inode should have `inode_operations`, to operate on the inode itself,
  and `file_operations`, to operate on the opened files
  - different types of inodes have different ops
    - `S_ISDIR`
    - `S_ISREG`
    - `S_ISLNK`
    - `S_ISCHR`
    - `S_ISBLK`
    - `S_ISFIFO`
    - `S_ISSOCK`
  - the root inode is a `S_ISDIR`
