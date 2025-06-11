VFS Mount
=========

## Initialization

- `mnt_init` creates `fs_kobj`, `/sys/fs`
  - `/sys/fs/bpf` is from `bpf_init`
  - `/sys/fs/cgroup` is from `cgroup_init`
  - `/sys/fs/ext4` is from `ext4_init_sysfs`
  - `/sys/fs/fuse` is from `fuse_sysfs_init`
  - `/sys/fs/pstore` is from `pstore_init_fs`
  - `/sys/fs/resctrl` is from `rdtgroup_init`
  - `/sys/fs/virtiofs` is from `virtio_fs_sysfs_init`

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

## kernel internals

- `path_mount` implements `mount` syscall, after copying strings from
  userspace and resolving the target path
  - depending on the flags, it performs one of these operations
    - if `MS_REMOUNT | MS_BIND`, `do_reconfigure_mnt`
    - if `MS_REMOUNT`, `do_remount`
    - if `MS_BIND`, `do_loopback`
    - if propagation type change, `do_change_type`
    - if `MS_MOVE`, `do_move_mount_old`
    - else `do_new_mount`
- `do_new_mount`
  - `get_fs_type` looks up `struct file_system_type` from the `fstype` string
    - we will use `ext4` as an example
  - `fs_context_for_mount` returns a transient `struct fs_context` whose
    lifetime is the same as the mount operation
    - the fs context `ops` is set by the fs type, such as `ext4_context_ops`
  - `vfs_get_tree` calls `ops->get_tree` which is expected to set `fc->root`
    - `ext4_get_tree` uses `get_tree_bdev` helper
    - `get_tree_bdev`
      - looks up `struct block_device` for the source
      - calls `sget_fc` to create `struct super_block`
      - initializes the super block using fs callback such as
        `ext4_fill_super`
        - it reads superblock data from the block device
        - it sets `struct super_operations` to manipulate the fs
        - it looks up the special root inode (`EXT4_ROOT_INO`) and wraps it in
          a `struct dentry`
      - points `fc->root` to the root dentry
  - `vfs_create_mount` creates a `struct vfsmount` from the root dentry of the
    source fs
  - `lock_mount` creates a `struct mountpoint` from the target path
  - `do_add_mount` attaches the mount to the mountpoint
