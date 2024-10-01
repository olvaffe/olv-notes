IOCTL
=====

## Format

- bit[7:0]: nr
  - ioctl number
- bit[15:8]: type
  - `DRM_IOCTL_BASE`: `d`
- bit[29:16]: size
  - `sizeof(struct)`
- bit[31:30]: dir
  - `_IOC_NONE`: 0
  - `_IOC_WRITE`: 1
  - `_IOC_READ`: 2

## Example

- `DRM_IOCTL_XE_EXEC` expands to
  `DRM_IOW(DRM_COMMAND_BASE + DRM_XE_EXEC, struct drm_xe_exec)`
- `DRM_IOW(nr,type)` expands to `_IOW(DRM_IOCTL_BASE,nr,type)`
  - `type` is the struct
- `_IOW(type,nr,size)` expands to
  `_IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))`
  - `type` is the ioctl type
  - `size` is the struct
  - `_IOC_TYPECHECK(t)` expands to `(sizeof(t))`
- `_IOC(dir,type,nr,size)` shifts and ors the fields
