DRM ioctl
=========

## Security

- any user in `video` group can open the primary device
- the user becomes master automatically if the primary device has no user
  - don't rely on this to run Xorg rootless
  - imagine VT swtiches, rootless Xorg cannot drop/acquire master
- any user can open the render node
- root-only ops
  - `DRM_IOCTL_SET_MASTER`
  - `DRM_IOCTL_DROP_MASTER`
- master-only ops
  - modesetting ioctls
- authenticated-only ops
  - `DRM_IOCTL_GEM_FLINK`
  - `DRM_IOCTL_GEM_OPEN`
  - not used in DRI3
- rendernode-allowed ops
  - `DRM_IOCTL_PRIME_HANDLE_TO_FD`
  - `DRM_IOCTL_PRIME_FD_TO_HANDLE`
  - `DRM_IOCTL_GEM_CLOSE`
  - and driver-specific execbuffer and alloc ops
  - rendernode-allowed ops must be explicitly whitelisted
  - while rendernode is assumed authenticated, the autenticated-only ops above
    are not on the white list
- `drm_ioctl_kernel` and ioctl flags
  - `DRM_ROOT_ONLY`
    - allow only if `capable(CAP_SYS_ADMIN)`
  - `DRM_AUTH`
    - allow only if `drm_is_render_client(file_priv)` or
      `file_priv->authenticated`
    - `authenticated` if root or master or authed by master
  - `DRM_MASTER`
    - allow only if `drm_is_current_master(file_priv)`
  - `DRM_RENDER_ALLOW`
    - when cleared, deny if `drm_is_render_client(file_priv)`
  - `DRM_UNLOCKED`
    - when cleared, lock `drm_global_mutex` if `DRIVER_LEGACY`
- ioctl hex val
  - according to `_IOC(dir,type,nr,size)` macro,
    - bit 0..7: `nr`
    - bit 8..15: `type`
      - always `DRM_IOCTL_BASE` (0x64) for drm ioctls
    - bit 16..29: `size`
      - `sizeof(arg)`
    - bit 30..31: `dir`
      - `_IOC_WRITE` is 1
      - `_IOC_READ` is 2
  - `DRM_IOCTL_GEM_CLOSE` is `0x40086409`
    - it expands to `DRM_IOW(0x09, struct drm_gem_close)`
    - `nr` is `0x09`
    - `type` is `0x64`
    - `size` is `sizeof(struct drm_gem_close)` which is 8
    - `nr` is `0x01` which is `_IOC_WRITE`

## dumb

- `DRM_IOCTL_MODE_CREATE_DUMB`
  - in: width, height, bpp
  - out: handle, pitch, size
- `DRM_IOCTL_MODE_ADDFB`
  - in: width, height, bpp, pitch, depth
- `DRM_IOCTL_MODE_MAP_DUMB` return an fake offset for mmap
  - for software rendering
- `DRM_CAP_DUMB_BUFFER` checks if the driver supports dumb
- `DRM_CAP_DUMB_PREFERRED_DEPTH` returns the preferred depth
  - static, usually 24 (because alpha is ignored)
  - 0 on virtio-gpu
- `DRM_CAP_DUMB_PREFER_SHADOW` returns whether shadow is preferred
  - static, usually true (because dumb might be in vram and/or uncached)
  - false on virtio-gpu
- xorg-modesetting without gbm does
  - when preferred depth is 8 or 16, depth=16, bpp=16, kbpp=0
  - otherwise,
    - depth=24, bpp=32, kbpp=0 if bpp=32 dumb and fb is supported
    - depth=24, bpp=32, kbpp=24 and forces bpp=32 shadow
- i915 assumes the dumb bo format to be `DRM_FORMAT_C8`, `DRM_FORMAT_RGB565`,
  or `DRM_FORMAT_XRGB8888` depending on bpp
