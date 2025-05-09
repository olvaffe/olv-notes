Kernel DRM Client
=================

## DRM Client

- all drivers should call `drm_client_setup` to set up an in-kernel client
  - it calls one of `drm_fbdev_client_setup` or `drm_log_register`, depending
    on `CONFIG_DRM_CLIENT_DEFAULT`
- a `drm_client_dev` is an in-kernel user of a drm device
- `drm_client_init` inits a `drm_client_dev` with a `drm_client_funcs`
  - it requires the drm dev to support `DRIVER_MODESET` and `dumb_create`
  - `drm_client_modeset_create` creates `client->modesets`
    - there is a `drm_mode_set` for each crtc
   - `drm_client_open` opens the primary node as `client->file`
- `drm_client_register` register a `drm_client_dev`
  - this add the client to `dev->clientlist` and calls
    `client->funcs->hotplug` to simulate a display hotplug event
- `drm_client_funcs`
  - `unregister` is called from `drm_dev_unregister` when the device is
    removed
  - `restore` is called from `drm_lastclose` when the last userspace client
    closes the file
  - `hotplug` is called from `drm_helper_hpd_irq_event` and the like on
    hotplug irq
  - `suspend` is called from `drm_mode_config_helper_suspend` when the device
    suspends
  - `resume` is called from `drm_mode_config_helper_resume` when the device
    resume
- utility functions
  - `drm_client_framebuffer_create` allocs a bo and an fb
    - `drm_mode_create_dumb` allocs the dumb bo
    - `drm_client_buffer_addfb` calls `drm_mode_addfb2` to create an fb
  - `drm_client_buffer_vmap` maps an bo (for sw rendering)
  - `drm_client_modeset_probe`
    - it collects all connectors, their modes, and their connection states
    - `drm_client_firmware_config` tries to inherit bios settings
    - it then saves the results to `client->modesets`
  - `drm_client_modeset_commit` loops `client->modesets` and sets all crtcs
  - `drm_client_modeset_dpms` disables crtcs

## fbdev client

- `drm_fbdev_client_setup`
  - it allocs a `drm_fb_helper`
  - `drm_fb_helper_prepare` inits the struct
  - `drm_client_init` inits `fb_helper->client` with `drm_fbdev_client_funcs`
  - `drm_client_register` registers `fb_helper->client`
- `drm_fbdev_client_funcs`
  - on restore, `drm_fb_helper_lastclose` restores fbdev modeset
    - e.g., when userspace compositor exits
  - on hotplug, there are two cases
    - on first hotplug, `drm_fb_helper_init` inits and
      `drm_fb_helper_initial_config` modesets
    - on subsequent hotplugs, `drm_fb_helper_hotplug_event` modesets
  - on suspend/resume, `drm_fb_helper_set_suspend_unlocked` informs fbcon
