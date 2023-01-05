Kernel open
===========

## chroot

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
