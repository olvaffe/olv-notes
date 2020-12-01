VC4 / V3D
=========

## VC4 and V3D

- `v3d_platform_driver` is compatible with NO known device
  - it is only used by downstream kernel...
- `vc4_platform_driver`, on the other hand, is compatible with
  - `brcm,bcm2835-vc4`
  - `brcm,bcm2711-vc5`
- `vc4_platform_driver` also has many component drivers
  - `vc4_hdmi_driver`, compatible with
    - `brcm,bcm2835-hdmi`
    - `brcm,bcm2711-hdmi0`
    - `brcm,bcm2711-hdmi0`
  - `vc4_vec_driver`, compatible with
    - `brcm,bcm2835-vec`
  - `vc4_dpi_driver`, compatible with
    - `brcm,bcm2835-dpi`
  - `vc4_dsi_driver`, compatible with
    - `brcm,bcm2835-dsi1`
  - `vc4_hvs_driver`, compatible with
    - `brcm,bcm2835-hvs`
    - `brcm,bcm2711-hvs`
  - `vc4_txp_driver`, compatible with
    - `brcm,bcm2835-txp`
  - `vc4_crtc_driver`, compatible with
    - `brcm,bcm2835-pixelvalve0`
    - `brcm,bcm2835-pixelvalve1`
    - `brcm,bcm2835-pixelvalve2`
    - `brcm,bcm2711-pixelvalve0`
    - `brcm,bcm2711-pixelvalve1`
    - `brcm,bcm2711-pixelvalve2`
    - `brcm,bcm2711-pixelvalve3`
    - `brcm,bcm2711-pixelvalve4`
  - `vc4_v3d_driver`, compatible with
    - `brcm,bcm2835-v3d`
- RPi [1-3] uses BCM285[5-7]
  - vc4 can fully support them
- RPi 4 uses BCM2711
  - vc4 is KMS only
  - no rendering nor rendernode
  - downstream kernel uses v3d for renderering

## Case Study: Sway

- when using a downstream kernel with vc4 and v3d, sway reports
  - Found 2 GPUs
  - Initializing DRM backend for /dev/dri/card0 (vc4)
  - Using atomic DRM interface
  - Found 5 DRM CRTCs
  - Found 26 DRM planes
  - EGL 1.4 / GLES 3.1 on Mesa 20.2.3
  - GL renderer: V3D 4.2
  - Initializing DRM backend for /dev/dri/card0 (vc4)
  - DRM universal planes unsupported
  - Failed to open DRM device 10
- it opens `/dev/dri/card0` for display
- it also create a `gbm_device` from the fd and create a
  `EGL_PLATFORM_GBM_MESA` EGL display
  - internally, Mesa calls `dri2_initialize_drm` to initialize the EGL display
    from the GBM device
  - `driver_name` is "vc4"
  - `is_render_node` is false
  - `/usr/lib/dri/vc4_dri.so` is dlopen'ed
  - however, `vc4_drm_screen_create` tests `DRM_VC4_PARAM_V3D_IDENT0` to see
    if the kernel vc4 driver supports v3d. If not, it falls back to
    `kmsro_drm_screen_create` which uses the first rendernode and its driver:
    v3d!
