coreutils
=========

## Overview

- <https://git.savannah.gnu.org/cgit/coreutils.git>
  - `[`, `test`
  - `basename`, `dirname`, `realpath`
  - `cat`, `head`, `cut`, `sort`, `tac`, `tail`, `tee`, `tr`, `uniq`, `wc`
  - `chmod`, `chown`, `chgrp`
  - `date`
  - `df`, `du`
  - `echo`, `printf`
  - `env`, `pwd`
  - `id`, `who`
  - `ln`, `mkdir`, `mkfifo`, `mknod`, `mktemp` `cp`, `dd`, `mv`, `rm`,
    `rmdir`, `touch`,
  - `ls`, `stat`, 
  - `nice`
  - `sha256sum`
  - `sleep`
  - `stty`, `tty`
  - `sync`
  - `uname`
  - etc.

## chmod

- special bits
  - `S_ISUID` (04000)  set-user-ID
    - set euid on `execve`
  - `S_ISGID` (02000)  set-group-ID
    - set egid on `execve`
    - new entries under the directory have the same gid
    - mandatory locking when `S_IXGRP` is cleared (do not use)
  - `S_ISVTX` (01000)  sticky bit
    - `unlink` additionally checks if file owner matches euid
- owner bits
  - `S_IRUSR` (00400)  read by owner
  - `S_IWUSR` (00200)  write by owner
  - `S_IXUSR` (00100)  execute by owner
  - for directories,
    - read is for `opendir`/`readdir`
    - write is for creating new entries or removing existing entries
    - execute is for accessing new/existing entries
- group bits
  - `S_IRGRP` (00040)  read by group
  - `S_IWGRP` (00020)  write by group
  - `S_IXGRP` (00010)  execute by group
- other bits
  - `S_IROTH` (00004)  read by others
  - `S_IWOTH` (00002)  write by others
  - `S_IXOTH` (00001)  execute by others
