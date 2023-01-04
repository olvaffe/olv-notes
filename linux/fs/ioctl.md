IOCTL
=====

## Format

- bit[7:0]: nr
  - ioctl number
- bit[15:8]: type
  - `DRM_IOCTL_BASE` 'd'
- bit[29:16]: size
  - sizeof(struct)
- bit[31:30]: dir
  - `_IOC_WRITE` 1
  - `_IOC_READ` 2
