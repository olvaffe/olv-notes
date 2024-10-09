Kernel DRM fbdev
================

## fbdev emulation

- a drm driver can call one of these to enable fbdev emulation
  - `drm_fbdev_dma_setup`
  - `drm_fbdev_ttm_setup`
  - `drm_fbdev_shmem_setup`
- during setup,
  - `drm_fb_helper_prepare` inits the `drm_fb_helper`
  - `drm_client_init` inits `fb_helper->client`
  - `drm_client_register` registers the client
    - this calls `drm_fbdev_dma_client_hotplug` for the first time
    - `drm_fb_helper_init` sets `dev->fb_helper` to the helper
    - `drm_fb_helper_initial_config`
      - `drm_client_modeset_probe` sets up crtc and connector
      - `drm_fb_helper_single_fb_probe` sets everything up
        - `drm_client_framebuffer_create` allocs a dumb bo and creates a
          `drm_framebuffer`
        - `drm_fb_helper_alloc_info` allocs a `fb_info` for fbdev
        - `drm_fb_helper_fill_info` fills in the `fb_info`
