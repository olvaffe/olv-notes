# Kernel namei

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
- `open` syscall as an example
  - `do_sys_open -> ... -> path_openat` looks up the dentry
  - `path_init`
    - `nd_jump_root` inits
      - `nd->root` to `fs->root`
      - `nd->path` to `nd->root`, which is `/`
      - `nd->inode` to `nd->path.dentry->d_inode`
  - `link_path_walk` has a loop to walk intermediate components
    - each loop iteration
      - update `nd->last` to the remaining components
        - `hash_name` returns the name with the current component skipped
      - `nd->last_type` is `LAST_NORM` (if the path is canonical)
      - if this is the last component, return
      - otherwise, `walk_component` walks the intermediate component
        - `lookup_slow` calls `inode->i_op->lookup` to lookup the child dentry
          under the parent inode
        - `step_into` updates `nd->path` and `nd->inode`
    - when `link_path_wlak` returns,
      - `nd->path` is the next-to-last component
      - `nd->last` is the last component
  - `open_last_lookups` looks up (and potentially opens) the last component
    - `lookup_open` looks up the dentry of the last component
    - `step_into` updates `nd->path` and `nd->inode`
  - `do_open` opens the last component
    - `may_open` checks permissions
    - `vfs_open` inits `file` from `inode`
      - `f->f_inode` is `inode`
      - `f->f_mapping` is `inode->i_mapping`
      - `f->f_op` is `inode->i_fop`
      - `f->f_op->open` opens the inode

## `d_path`

- `d_path` is the opposite of `filename_lookup`
- it is used to when kernel needs the filename of a dentry
  - `/proc/<pid>/fd`
  - `/proc/<pid>/maps`, indirecly via `seq_path`
  - etc.
- if the dentry is deleted, it appends ` (deleted)`
- if the (pseudo) fs provides `dentry_operations::d_name`, it returns the
  result instead
  - socket `sockfs_dname` returns `socket:[ino]`
  - pipe `pipefs_dname` returns `pipe:[ino]`
  - dmabuf `dmabuffs_dname` returns `/dmabuf:<name>`, where name is from
    `DMA_BUF_SET_NAME` ioctl
  - anon inode `anon_inodefs_dname` returns `anon_inode:<name>`, where name is
    specified by `anon_inode_getfile`, etc.
  - simple `simple_dname` returns `/<name> (deleted)`, where name is specified
    by `alloc_file_pseudo`
