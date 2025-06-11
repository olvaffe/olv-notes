Linux VFS xattr
===============

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
