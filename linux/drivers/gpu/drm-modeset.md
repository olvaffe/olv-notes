DRM Modesetting
===============

## DRM modifiers

- To kernel,
  - `DRM_FORMAT_MOD_INVALID` (0x00ffffffffffffff) is truely invalid
  - `DRM_FORMAT_MOD_LINEAR` (0x0) is truely linear
  - in addfb2, when `DRM_MODE_FB_MODIFIERS` is not set, modifiers are not
    (explicitly) given.  modifiers array are required to be zeroed, but it is
    not the same as `DRM_FORMAT_MOD_LINEAR`
    - `intel_framebuffer_init` changes modifiers[0] when not explicitly given
      before calling `drm_helper_mode_fill_fb_struct` to set `fb->modifier`
  - a plane's `IN_FORMATS` never contains `DRM_FORMAT_MOD_INVALID`
- To EGL,
  - in eglCreateImage, when modifier is not explicitly given, it is considered
    to be `DRM_FORMAT_MOD_INVALID` and means impl-defined (similiar to addfb2
    without `DRM_MODE_FB_MODIFIERS`)
  - however, when `DRM_FORMAT_MOD_INVALID` is explicitly given, it becomes
    invalid
- xorg-modesetting allocates front buffers that are rendered to by glamor and
  scanned out by display engine
  - traditionally, set `GBM_BO_USE_RENDERING | GBM_BO_USE_SCANOUT` and be done
  - when supported, use modifiers instead
  - during crtc init, xorg-modesetting finds the best plane for the crtc
    - `drmmode_crtc_create_planes` also decodes the `IN_FORMATS` prop of the
      plane to get the list of supported formats and modifiers
    - virtio-gpu supports two planes
      - primary supports XR24
      - cursor supports AR24
    - i915 supports 9 planes
      - three primary supports CB RG16 XR24 XB24 XR30 XB30 XB4H (LINEAR or x-tiled)
      - three overlay supports XR24 XB24 XR30 XB30 XR4H XB4H YUYV YVYU UYVY VYUY (LINEAR or x-tiled)
      - three cursor supports AR24 (LINEAR)
  - bo allocation will pass in the modifiers supported by the crtc/plane
- glamor needs to allocate a pixmap or dmabuf-to-pixmap or pixmap-to-dmabuf
  - for alloc, `glTexImage2D`
  - for import, `gbm_bo_import`, `eglCreateImageKHR`, and
    `glEGLImageTargetTexture2DOES`
  - for export, `gbm_bo_create*`, import as pixmap, CopyArea, and replace the
    texture-based pixmap by the bo-based pixmap
  - EGL dmabuf extensions allow glamor to know which modifiers are supported
- gbm supports usage or modifiers, but not both
  - for usage, the dri backend only cares about `GBM_BO_USE_SCANOUT`,
    `GBM_BO_USE_CURSOR`, and `GBM_BO_USE_LINEAR`
  - i965's `intel_create_image_common` uses usage to pick between linear and
    x-tiled; for modifiers, it becomes explicit and can fail (when the
    modifier is not supported)
  - gallium drivers must implement `resource_create_with_modifiers` to support
    modifiers
  - gallium's `dri2_create_image_common` always sets `PIPE_BIND_RENDER_TARGET`
    and `PIPE_BIND_SAMPLER_VIEW`; it uses usage to set scanout/linear/cursor
    binds, which are not set with modifiers
- EGL supports modifiers
  - when importing a dma-buf, no modifier means implementation-defined
    modifier; `DRM_FORMAT_MOD_INVALID` is not a valid explicit modifier

## DRM formats

- `DRM_IOCTL_MODE_ADDFB2` takes a format instead of bpp/depth.  It also
  supports modifiers and planar formats.
  - internally, `drm_mode_legacy_fb_format` translates bpp/depth to format and
    `DRM_IOCTL_MODE_ADDFB` can take the same path as addfb2 does
- modifiers must be explicitly enabled by setting `DRM_MODE_FB_MODIFIERS` flag

## Modeset Fences

- explicit in-fencing
  - userspace sets the in-fence using `IN_FENCE_FD` property
  - `drm_atomic_plane_set_property` saves the fence at `state->fence`
  - `drm_gem_plane_helper_prepare_fb` merges `DMA_RESV_USAGE_KERNEL` implicit
    fences associated with fbs, if any, to `state->fence`
    - if not doing explicit in-fencing, this merges `DMA_RESV_USAGE_WRITE`
      implicit fences instead
  - `drm_atomic_helper_wait_for_fences` waits on `state->fence`
  - commit!
- explicit out-fencing
  - userspace sets the out-fence using `OUT_FENCE_PTR` property
  - `drm_atomic_crtc_set_property` saves the ptr to
    `state->crtcs[idx].out_fence_ptr`
  - `prepare_signaling` calls `drm_crtc_create_fence` to create a new fence
    and calls `setup_out_fence` to point the out-fence to the new fence
    - it also calls `create_vblank_event` to be notified on next vsync
    - there are also writeback fences associated with connectors, which will
      be signaled by `drm_writeback_signal_completion`
  - `complete_signaling` returns the out fences as fds
  - on next vblank, `drm_crtc_send_vblank_event` calls `drm_send_event_helper`
    indirectly to signal the out-fence
  - note that when userspace makes a commit, the new fbs will replace the
    current fbs on next vblank and the out-fence will signal
    - the current fbs (instead of the new ones) become available when the
      out-fence signals
  - there is implicit in-fencing but no implicit out-fencing
    - the userspace compositor traditionally requests a
      `DRM_MODE_PAGE_FLIP_EVENT` to get notified
