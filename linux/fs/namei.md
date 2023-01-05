Kernel namei
============

## Path Resolution

- <Documentation/filesystems/path-lookup.rst>
  - namei stands for name-to-inode
- the goal is use a filename to find the `struct dentry` from dcache
- `getname` allocates a `struct filename` to hold the filename
  - this copies the name from the userspace pointer
- `filename_lookup` converts the filename to a `struct path`
  - `path_lookupat` has a tight loop to call `link_path_walk` and
    `lookup_last`
  - `struct path` contains `struct vfsmount` and `struct dentry`
- `statx` syscall as an example
  - `strace stat /` shows that the command uses `statx` syscall
  - it uses `filename_lookup` to get `path`
  - `path` has a pointer to `dentry` which has a pointer to `struct inode`
  - `vfs_getattr_nosec` reads the attrs from the inode
