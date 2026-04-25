# Linux VFS IO

## Overview

- `vfs_read` calls `file->f_op->read_iter`
  - `init_sync_kiocb` wraps a file
  - `iov_iter_ubuf` wraps a buf
