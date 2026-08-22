# Xwayland

## Build

- build
  - `meson out -Dxorg=false -Dxnest=false -Dxvfb=false -Dxwayland=true -Dglamor=true`
- without glamor, `wl_shm` is used
- with glamor, `wl_drm` and/or `zwp_linux_dmabuf_v1` are used
  - `xwl_glamor_init_backends` initializes backends supported by EGL
    - it does not communicate with the server yet
  - `wl_registry_add_listener` and `xwl_screen_roundtrip` initializes the
    global objects
    - it keeps making roundtrips until there is no event
  - `xwl_glamor_select_backend` selects the backend
  - `xwl_glamor_init` initializes the backend
    - `xwl_glamor_gbm_init_egl`
    - `glamor_init`
    - `xwl_glamor_gbm_init_screen`
- when xwayland flips in `xwl_present_flip`, it calls
  - `wl_surface_attach` to attach the new dma-buf
  - `wl_surface_frame` to request a frame callback
  - `wl_surface_damage_buffer` to damage
  - `wl_surface_commit` to commit
  - `wl_display_flush` to flush
  - after a while, the frame callback is received
    - this happens when wayland is ready for another `wl_surface_commit`
    - xwayland calls `xwl_present_frame_callback` to send
      `PresentCompleteNotify` to the client and to execute another flip
