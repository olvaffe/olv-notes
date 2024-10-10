Kernel VFS sync
===============

## Syscalls

- `SYSCALL_DEFINE1(fsync, unsigned int, fd)` makes sure all changes have
  reached the permanent storage device
  - it transfers all modified buffer cache pages (data and metadata) to the
    device, waits for the transfers to finish, and flushes the device cache
    - watch out for buggy fs that does not flush device cache
    - watch out for buggy disk firmware...
  - if this is a new file in a directory, fsync on the directory is also
    needed
- `SYSCALL_DEFINE1(fdatasync, unsigned int, fd)` is similar to `fsync(fd)`
  except that it does not sync non-essential metadata such as `st_atime` or
  `st_mtime`
- `SYSCALL_DEFINE1(syncfs, int, fd)` transfers all modifications to the
  filesystem the file resides in, waits for transfers to finish, and flushes
  the device cache.
  - linux specific
- `SYSCALL_DEFINE0(sync)` "schedules" all modifications to all filesystems to
  be transferred
  - that is what POSIX says
  - linux waits and flushes
- `sync` command
  - `sync <file>` does `fsync()`
  - `sync --data <file>` does `fdatasync()`
  - `sync --file-system <file>` does `syncfs()`
  - `sync` or `sync --file-system` does `sync()`
- conclusion
  - on Linux, `sync` is enough
  - watch out for buggy fs
  - watch out for buggy disk controller
