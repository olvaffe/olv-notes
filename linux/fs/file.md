Linux VFS File
==============

## Overview

- a `struct file` is a file (that is opened)
  - `file->f_inode` is non-NULL
- to associate a file with an fd,
  - `get_unused_fd_flags` allocs an unused index from `current->files->fdt`
  - `fd_install` sets `current->fileds->fdt->fd[fd]` to the file
    - this transfers ownership
