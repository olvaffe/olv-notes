Kernel DRM Client
=================

## DRM Client

- a `drm_client_dev` is an in-kernel user of a drm device
  - it is used by fbdev emulation, which is an in-kernel user of a drm device
- `drm_client_init` inits a `drm_client_dev`
  - it requires the drm dev to support `DRIVER_MODESET` and `dumb_create`
  - `drm_client_modeset_create` creates `client->modesets`
    - there is a `drm_mode_set` for each crtc
   - `drm_client_open` opens the primary node as `client->file`
- `drm_client_register` register a `drm_client_dev`
  - this add the client to `dev->clientlist` and calls
    `client->funcs->hotplug` to simulate a display hotplug event
- on display hotplug, fbdev emu calls
  - `drm_client_modeset_probe`
    - it collects all connectors, their modes, and their connection states
    - `drm_client_firmware_config` collects crtcs, display native modes, etc.
    - it then saves the results to `client->modesets`
  - `drm_client_modeset_commit` loops `client->modesets` and sets all crtcs
- `drm_client_modeset_dpms` disables crtcs
  - it is called from `drm_client_modeset_dpms`, which is not sued by generic
    fbdev emu
- `drm_client_framebuffer_create` allocs a bo and an fb
  - `drm_mode_create_dumb` allocs the dumb bo
  - `drm_client_buffer_addfb` calls `drm_mode_addfb2` to create an fb
- `drm_client_buffer_vmap` maps an bo (for sw rendering)
