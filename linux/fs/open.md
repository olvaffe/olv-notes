Kernel open
===========

## `SYSCALL_DEFINE1(chroot)`

- the userspace chroot command makes 3 syscalls
  - `chroot("somewhere")`
  - `chdir("/")`
  - `execve("/bin/bash", ["/bin/bash", "-i"], ...)`
- `current->fs` is a `struct fs_struct`
  - `int umask`
  - `struct path root`
  - `struct path pwd`
- `chroot` syscall
  - `user_path_at` resolves the filename to `path`
    - `path` holds `dentry` which holds `inode`
  - `set_fs_root` updates `current->fs->root`
- `chdir` syscall
  - `user_path_at` resolves the filename to `path`
  - `set_fs_pwd` updates `current->fs->pwd`

## `SYSCALL_DEFINE2(ftruncate)`

- `do_truncate` calls `notify_change` to set `ATTR_SIZE` to the new size

## `SYSCALL_DEFINE4(fallocate)`

- the underlying fs must provide `file->f_op->fallocate`
- default op preallocates blocks for `[offset, offset+size)`
  - if blocks are already allocated, nop
  - otherwise, it preallocates blocks and zeros them
  - advanced fs can preallocate "zero blocks" to avoid io
- `FALLOC_FL_ZERO_RANGE` preallocates and zeros blocks for `[offset, offset+size)`
  - it differs from default op in that, if blocks are already allocated, they
    are zeroed
  - advanced fs can use "zero blocks" to avoid io
- `FALLOC_FL_PUNCH_HOLE` deallocates blocks for `[offset, offset+size)`
  - it deallocates blocks and the range reads back as zero
  - advanced fs uses "zero blocks" instead of deallocating them
- the ops affect both `st_size` and `st_blocks`
  - `FALLOC_FL_KEEP_SIZE` keeps `st_size` unchanged
  - in contrast, `ftruncate` affects only `st_size` (when the fs supports
    sparse)
