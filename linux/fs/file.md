# Linux VFS File

## Overview

- a `struct file` represents an opened instance of an inode
  - `file->f_inode` is non-NULL
  - `file->f_path` is the full path, including mount and dentry
  - there is `struct file_operations`
- to associate a file with an fd,
  - `get_unused_fd_flags` allocs an unused index from `current->files->fdt`
  - `fd_install` sets `current->fileds->fdt->fd[fd]` to the file
    - this transfers ownership
  - two fds might point to the same file (e.g., dup)
