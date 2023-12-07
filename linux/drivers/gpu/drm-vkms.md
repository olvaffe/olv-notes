DRM vkms
========

## vkms

- `vkms_driver`
  - `driver_featues` is `DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM`
    - `DRIVER_MODESET` supports all modeset ioctls
    - `DRIVER_ATOMIC` supports all atomic modeset ioctls
    - `DRIVER_GEM` is set for all modern drivers
  - `fops` is `DEFINE_DRM_GEM_FOPS`
    - all fops (open, close, ioctl, poll, read, mmap) are standard
  - prime and dumb are `DRM_GEM_SHMEM_DRIVER_OPS`
  - no fencing
- kms
  - `drm_vblank_init` initializes `dev->vblank`
  - `vkms_modeset_init` initializes `dev->mode_config`
  - `vkms_output_init` initializes `vkmsdev->output`
    - one `drm_crtc`
      - this allocates a fence context for OUT-fences
    - one `DRM_PLANE_TYPE_PRIMARY` `drm_plane`
    - one `drm_connector`
    - one `drm_encoder`
    - and optionally,
      - multiple `DRM_PLANE_TYPE_OVERLAY` `drm_plane`
      - one `DRM_PLANE_TYPE_CURSOR` `drm_plane`
      - one `drm_writeback_connector`
        - this allocates a fence context for OUT-fences
- modes
  - `drm_add_modes_noedid` is called to add all modes below
    `XRES_MAX / YRES_MAX` (8192x8192)
  - the preferred mode is `XRES_DEF / YRES_DEF` (1024x768)
- vblank
  - vblank is simulated with a hrtimer
  - `period_ns` is from `framedur_ns`, which is calculated from the current
    mode using `drm_calc_timestamping_constants`

## vkms usecases

- kms testing
  - igt can exercise kms paths using vkms
- compositor testing
  - compositors can run on vkms
- running compositors headless
  - useful for compositors that cannot support headless otherwise
