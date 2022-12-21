# VFS

## Concepts

- a filesystem consists of tons of objects (files, directories, fifos, etc.)
  and a superblock (metadata) to help manage the objects
  - a `struct super_block` is a in-memory representation of the filesystem
    superblock
  - a `struct inode` is a in-memory reprentation of a filesystem object
  - a pathname is a key to look up an object from the super block
  - a soft link is a separate inode (an seprate fs object); a hard link is the
    same inode (not an seprate fs object)
- to look up /for/example,
  - after the filesystem is mounted, we have the inode for "/"
  - we look up the inode for "for" under "/"
  - we look up the inode for "example" under "/foo"
- dcache speeds up lookups
  - once an inode is looked up, we set up a `struct dentry` with
    - `d_parent` points to the parent dentry
    - `d_subdirs` is a list of children dentry
    - `d_name` equal to the name
    - `d_inode` points to the inode
- page cache speeds up read/write
  - each inode has a `i_mapping` pointing to its own `i_data`, which is a
    `struct address_space`
  - given a `pgoff_t`, allocate a page, read the object at the offset into
    the page, and add the page into `address_space`

## Introduction

- VFS provides directory entry cache (aka dentry cache or dcache) to map a
  pathname to a dentry.  Dentries live in RAM and are never saved to the fs
  storage device: they exist for performance.
- A dentry usually has a pointer to an indoe.  Inodes are fs objects such as
  files, directories, FIFOs, etc.  They live on the fs storage device.  They
  are copied into RAM when required, and changes to the copy are written back
  to the storage device.
  - Multiple dentries may point to the same inode (e.g., hardlinks)
- Opening a file allocates a `struct file` which has a pointer to the dentry
  - it is placed in the process's file descriptor table
  - two fds might point to the same file (e.g., dup)

## Registering and Mounting a fs

- A fs is registered with `register_filesystem(struct file_system_type *);`
  - /proc/filesystems lists the registered filesystems
- `file_system_type::mount` is called when a new instance of this fs should be
  mounted
  - it is given a device name, which is the device the fs instance lives on
  - it creates and initializes a `struct super_block` and returns a dentry to
    the root directory (or a subtree)
- `file_system_type::kill_cb` is called when a new instance of this fs should be
  shut down

## Superblock

- A superblock object represents a mounted filesystem
- there is `struct super_operations` that tells the VFS how to manipulate the
  filesystem
  - there is an `alloc_inode` method to allocate and initialize `struct inode`

## Inode

- An inode object represents an object within the fs
- there is `struct inode_operations` that tells the VFS how to manipulate the
  object, such as create/lookup/link/unlink/...

## Address Space

- `struct address_space` is used to group and manage pages in the page cache.
  It track the pages of an inode.
- there is `struct address_space_operations`

## File

- A file object represents an opened instance of an inode
- there is `struct file_operations`

## Directory Entry Cache (dcache)

## `anon_inodefs`

- during kernel init, `anon_inode_init` mounts the filesystem
  - it also calls `alloc_anon_inode` to allocate an inode
    - the function assigns the inode a unique ino allocated by `get_next_ino`
  - this is the only inode in the filesystem
- `anon_inode_getfile` opens the only inode and returns a file
  - it calls `alloc_file_pseudo`, which
    - calls `d_alloc_pseudo` to create a dentry with the given name
    - calls `alloc_file` to create a file and install the given fops

## `mount`

- the `mount` syscall takes
  - `source` specifies a fs to be attached
    - a string to dev, file, dir, or just a dummy one
  - `target` specifies the location to attach the fs to
    - a string to a file or dir
  - `filesystemtype` specifies the type of the fs
    - a string whose valid values are in `/proc/filesystems`
  - `mountflags`
  - `data` specifies fs-specific data
    - usually a string of comma-separated fs-specific options
- a call to `mount` conducts a sequence of operations in order
  - if `MS_REMOUNT`, remount an existing mount
  - if `MS_BIND`, create a bind mount
  - if `MS_SHARED`/`MS_PRIVATE`/`MS_SLAVE`/`MS_UNBINDABLE`, change the
    propagation type of an existing mount
  - if `MS_MOVE`, move an existing mount
  - if none of the above, create a new mount
- remount an existing mount
  - `source` and `filesystemtype` are ignored
  - `target` specifies the existing mount
  - `mountflags` and `data` can be modified
- create a bind mount
  - `source` and `target` can be file or dir
  - when a dir is bind-mounted, only the dir is mounted
    - submounts under the dire is not bind-mounted
    - `MS_REC` to bind-mound submounts recursively
- changing the propagation type of an existing mount
  - `source`, `filesystemtype`, and `data` are ignored
  - `target` specifies the existing mount
  - only one of the propagation types can be specified
    - `MS_SHARED`
    - `MS_PRIVATE`
    - `MS_SLAVE`
    - `MS_UNBINDABLE`
  - by default, only the mount is affected
    - set `MS_REC` to affect submounts
- move an existing mount
  - `source` specifies the existing mount
  - `target` specifies the new location
  - `filesystemtype`, `mountflags` (execept for `MS_MOVE` of course) and
    `data` are ignored
- create a new mount
  - all arguments are used
- other flags
  - `MS_DIRSYNC` makes dir change synchronous
  - `MS_LAZYTIME` flushes atime/mtime/ctime changes to device lazily
  - `MS_MANDLOCK` permits mandatory file locking
  - `MS_NOATIME` stops updating atime
  - `MS_NODEV` disallows dev special files
  - `MS_NODIRATIME` stops updating atime for dirs
  - `MS_NOEXEC` disallows executions
  - `MS_NOSUID` ignores suid
  - `MS_RDONLY` mounts read-only
  - `MS_REC` is for use with bind mount and propagation type change
  - `MS_RELATIME` updates atime when it is older than mtime/ctime
    - this is the default behavior
  - `MS_SILENT` suppresses some printks
  - `MS_STRICTATIME` always updates atime on access
  - `MS_SYNCHRONOUS` makes all writes synchronous
  - `MS_NOSYMFOLLOW` stops following symlinks
- peer group example
  - on boot, all regular mounts are `MS_SHARED` and belong to different peer
    groups
  - `unshare -m --propagation unchanged`
    - this creates a new mount namespace with all mounts and all propagation
      types replicated
    - all mounts in the new namespace belong to the same peer groups in the
      original namespace
  - `mount /dev/foo /mnt` in the either namespace shows up in another
  - in other words, when a submount (`/mnt` in this example) is
    created/destroyed, the parent mount (`/`) receives the event
    - because `/` is shared in either namespace, it shares the event to its
      peer group
- peer groups
  - `man mount_namespaces`
  - all shared/slave mounts are members of peer groups
    - a shared mount has `shared:X` in `cat /proc/self/mountinfo`
      - `X` is the peer group id
    - a slave mount has `master:X` instead
  - peer group creation
    - when a mount is created, and when it is shared, it becomes the sole
      member of a newly created peer group
      - this commonly happens when the mount's parent mount is shared
    - when a private mount is made shared, it also becomes the sole member of
      a newly created peer group
  - peer group destruction
    - when the last mount in a peer group is destroyed, the peer group is
      destroyed
  - how does a mount join a peer group?
    - when a mount is a bind-mount, and the source mount is shared, the
      bind-mount joins the peer group of the source mount
    - when a mount is replicated in a new mount namespace, the mount joins the
      peer group of the original mount
- event propagation
  - when a submount is created/destroyed, its parent mount receives the event
  - if the parent mount is shared, it shares the event to all peers in its
    peer group
  - if the parent mount is private, there is no peer group for it to share
    with
  - if the parent mount is slave, it does not share events
    - on the other hand, a peer that is shared will share events to this
      parent mount
