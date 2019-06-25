# VFS

## Introduction

* VFS provides directory entry cache (aka dentry cache or dcache) to map a
  pathname to a dentry.  Dentries live in RAM and are never saved to the fs
  storage device: they exist for performance.
* A dentry usually has a pointer to an indoe.  Inodes are fs objects such as
  files, directories, FIFOs, etc.  They live on the fs storage device.  They
  are copied into RAM when required, and changes to the copy are written back
  to the storage device.
  * Multiple dentries may point to the same inode (e.g., hardlinks)
* Opening a file allocates a `struct file` which has a pointer to the dentry
  * it is placed in the process's file descriptor table
  * two fds might point to the same file (e.g., dup)

## Registering and Mounting a fs

* A fs is registered with `register_filesystem(struct file_system_type *);`
  * /proc/filesystems lists the registered filesystems
* `file_system_type::mount` is called when a new instance of this fs should be
  mounted
  * it is given a device name, which is the device the fs instance lives on
  * it creates and initializes a `struct super_block` and returns a dentry to
    the root directory (or a subtree)
* `file_system_type::kill_cb` is called when a new instance of this fs should be
  shut down

## Superblock

* A superblock object represents a mounted filesystem
* there is `struct super_operations` that tells the VFS how to manipulate the
  filesystem
  * there is an `alloc_inode` method to allocate and initialize `struct inode`

## Inode

* An inode object represents an object within the fs
* there is `struct inode_operations` that tells the VFS how to manipulate the
  object, such as create/lookup/link/unlink/...

## Address Space

* `struct address_space` is used to group and manage pages in the page cache.
  It track the pages of an inode.
* there is `struct address_space_operations`

## File

* A file object represents an opened instance of an inode
* there is `struct file_operations`

## Directory Entry Cache (dcache)

