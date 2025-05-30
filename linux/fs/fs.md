VFS
===

## Filesystems

- bootloader
  - `CONFIG_FAT_FS` for uefi esp
- root
  - `CONFIG_EXT4_FS` the most common
  - `CONFIG_BTRFS_FS` more feature-rich
  - `CONFIG_BCACHEFS_FS` new and unstable
  - `CONFIG_XFS_FS` default on RHEL
  - `CONFIG_F2FS_FS` designed for ssd, emmc, and sd
  - `CONFIG_ZFS` out-of-tree
- read-only and overlay
  - `CONFIG_SQUASHFS` the most common
  - `CONFIG_EROFS_FS` faster, default on android
  - `CONFIG_CRAMFS` obsoleted, precursor to squashfs
  - `CONFIG_ROMFS_FS` too limited
  - `CONFIG_ISO9660_FS` cd
  - `CONFIG_OVERLAY_FS` write overlay
- mtd
  - `CONFIG_JFFS2_FS` the most common
  - `CONFIG_UBIFS_FS` newer and better
- network
  - `CONFIG_NFS_FS` the most common
  - `CONFIG_SMBFS` cross-platform file sharing
  - `CONFIG_9P_FS` vm file sharing
- windows
  - `CONFIG_EXFAT_FS` modern and cross-platform
  - `CONFIG_NTFS3_FS` read-write
  - `CONFIG_NTFS_FS` older, read-only, partial write
- osx
  - `CONFIG_APFS` out-of-tree
  - `CONFIG_HFSPLUS_FS` predate osx
- bsd
  - `CONFIG_UFS_FS` the most common
- encryption
  - `CONFIG_ECRYPT_FS` stacked, don't use
    - use fscrypt-enabled fs or dm-crypt instead

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

## Initialization

- `mnt_init` creates `fs_kobj`, `/sys/fs`
  - `/sys/fs/bpf` is from `bpf_init`
  - `/sys/fs/cgroup` is from `cgroup_init`
  - `/sys/fs/ext4` is from `ext4_init_sysfs`
  - `/sys/fs/fuse` is from `fuse_sysfs_init`
  - `/sys/fs/pstore` is from `pstore_init_fs`
  - `/sys/fs/resctrl` is from `rdtgroup_init`
  - `/sys/fs/virtiofs` is from `virtio_fs_sysfs_init`

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
- `statx` returns the attrs of a inode
- an inode can be in one of the following types
  _ `S_IFSOCK`, socket
  _ `S_IFLNK`, symbolic link
  _ `S_IFREG`, regular file
  _ `S_IFBLK`, block device
  _ `S_IFDIR`, directory
  _ `S_IFCHR`, character device
  _ `S_IFIFO`, FIFO
  - filesystems use different `inode_operations` for inodes of different types

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

## Extended Attributes

- tools
  - `setfacl` calls `setxattr` with `system.posix_acl_access`
  - `setcap` calls `setxattr` with `security.capability`
  - `setfattr` calls `setxattr` with the specified name
- `setxattr` syscall
  - if the name is `system.posix_acl_access`, `do_set_acl` calls `vfs_set_acl`
    which calls `inode->i_op->set_acl`
  - otherwise, `vfs_setxattr` calls `xattr_resolve_name` to resolve the name
    to a handler
- ext4
  - `CONFIG_EXT4_FS_POSIX_ACL` enables `ext4_set_acl` callback
  - `CONFIG_EXT4_FS_SECURITY` enables `ext4_xattr_security_handler` handler
