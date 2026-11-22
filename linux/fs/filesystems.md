# VFS

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

## `anon_inodefs`

- during kernel init, `anon_inode_init` mounts the filesystem
  - it also calls `alloc_anon_inode` to allocate an inode
    - the function assigns the inode a unique ino allocated by `get_next_ino`
  - this is the only inode in the filesystem
- `anon_inode_getfile` opens the only inode and returns a file
  - it calls `alloc_file_pseudo`, which
    - calls `d_alloc_pseudo` to create a dentry with the given name
    - calls `alloc_file` to create a file and install the given fops
