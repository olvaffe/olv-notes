wlroots
=======

## sway

- require `WAYLAND_DISPLAY=wayland-1`
  - `libwayland-client` defaults to `wayland-0`
  - we want to avoid that as explained in
    <https://gitlab.freedesktop.org/wayland/weston/-/merge_requests/486>

## running sway from ssh session

- `WLR_LIBINPUT_NO_DEVICES=1 WLR_SESSION=noop sway -d`

## `swaymsg -t get_tree`

- when there is an Alacritty window on the screen
  - node
    - `type`: `root`
    - `name`: `root`
    - node
      - `type`: `output`
      - `name`: `__i3`
      - node
        - `type`: workspace
        - `name`: `__i3_scratch`
    - node
      - `type`: `output`
      - `name`: `eDP-1`
      - node
        - `type`: `workspace`
        - `name`: `1`
        - node
          - `type`: `con`
          - `name`: current window title
          - `pid`: process pid
          - `app_id`: `Alacritty`
          - `shell`: `xdg_shell`

## Sway on X11

- `server_privileged_prepare`
  - it calls `wl_display_create` and `wl_display_get_event_loop` to create a
    `wl_display` and get its `wl_event_loop`
  - it then calls `wlr_backend_autocreate` to create a `wlr_backend` for the
    `wl_display`
- `server_init`
  - it calls `wlr_backend_get_renderer` and `wlr_renderer_init_wl_display`
    to initialize `wlr_renderer` with `wl_display`.  It creates these
    globals
    - `wl_display_init_shm` by libwayland-server
    - `wayland_drm_init` by Mesa
    - `wlr_linux_dmabuf_v1_create` by libwlroots
  - it creates these globals for the `wl_display`
    - `wlr_compositor_create`
    - `wlr_data_device_manager_create`
    - `wlr_gamma_control_manager_v1_create`
    - `wlr_xdg_output_manager_v1_create`
    - `wlr_idle_create`
    - `wlr_idle_inhibit_v1_create`
    - `wlr_layer_shell_v1_create`
    - `wlr_xdg_shell_create`
    - `wlr_tablet_v2_create`
    - `wlr_server_decoration_manager_create`
    - `wlr_xdg_decoration_manager_v1_create`
    - `wlr_relative_pointer_manager_v1_create`
    - `wlr_pointer_constraints_v1_create`
    - `wlr_presentation_create`
    - `wlr_output_manager_v1_create`
    - `wlr_output_power_manager_v1_create`
    - `wlr_input_method_manager_v2_create`
    - `wlr_text_input_manager_v3_create`
    - `wlr_foreign_toplevel_manager_v1_create`
    - `wlr_export_dmabuf_manager_v1_create`
    - `wlr_screencopy_manager_v1_create`
    - `wlr_data_control_manager_v1_create`
    - `wlr_primary_selection_v1_device_manager_create`
    - `wlr_viewporter_create`
  - it calls `wlr_noop_backend_create`
    - this is used only when there is no output
  - it calls `wlr_headless_backend_create_with_renderer`
    - this is used to virtual outputs
  - it calls `input_manager_create` which creates these globals
    - `wlr_virtual_keyboard_manager_v1_create`
    - `wlr_virtual_pointer_manager_v1_create`
    - `wlr_input_inhibit_manager_create`
    - `wlr_keyboard_shortcuts_inhibit_v1_create`
  - it calls `input_manager_get_seat` which creates these globals
    - `wlr_seat_create`
    - `wlr_pointer_gestures_v1_create`
- `server_start`
  - it calls `wlr_xwayland_create` unless xwayland is disabled
  - it calls `wlr_backend_start` to start the backend
    - this creates a X11 window and calls `wlr_x11_output_create` to create
    	a global
- `server_run`
  - this calls `wl_display_run` to enter the mainloop


## `wlr_backend_autocreate`

- this returns a `wlr_multi_backend`
  - a multi backend is a `wlr_backend` that contains a list of other backends
  - it dispatches all operations to the list of other backends
- On wayland, this calls `wlr_wl_backend_create` and `wlr_wl_output_create`
  - it connects to Wayland display and initializes the connection
  - `wlr_renderer_autocreate` is called with `EGL_PLATFORM_WAYLAND_EXT` to
    create a `wlr_renderer`
  - it gets the drm fd from the renderer and calls `wlr_gbm_allocator_create`
    to create a `gbm_device`
  - when it comes time to present, `output_commit` creates the `wl_buffer`s
    from dma-bufs exported from `gbm_bo`s
- On X11, this calls `wlr_x11_backend_create` and `wlr_x11_output_create`
  - it connects to X11 display and initializes the connection
  - it uses DRI3 to get the drm fd and calls `wlr_gbm_allocator_create` to
    create a `gbm_device`
  - `wlr_renderer_autocreate` is called with `EGL_PLATFORM_GBM_KHR` to create
    a `wlr_renderer`
  - when it comes time to present, `output_commit` creates the `xcb_pixmap_t`s
    from dma-bufs exported from `gbm_bo`s
- On DRM, this calls `wlr_libinput_backend_create` and
  `wlr_drm_backend_create`
  - with the help of `wlr_session` created by `wlr_session_create`
    - users normally don't have the privileges to open drm or evdev devices
    - when a session wraps a `logind` user session, opening is done by
      `logind`, which has the concept of seats and devices attached to seats
  - it opens the DRM node to get the drm fd and calls
    `wlr_gbm_allocator_create` to create a `gbm_device`
  - `wlr_renderer_autocreate` is called with `EGL_PLATFORM_GBM_KHR` to create
    a `wlr_renderer`
  - when it comes time to present, `drm_connector_commit` calls
    `drm_fb_import` to create DRM fbs from dma-bufs exported from `gbm_bo`s

## `wlr_renderer_autocreate`

- `wlr_egl_init` initializes `wlr_egl`
  - it uses the platform specified by the backend
  - `EGL_OPENGL_ES2_BIT` is appended to attrs
  - these extensions are required
    - `EGL_EXT_client_extensions`
    - `EGL_EXT_platform_base`
    - I believe things fail down later when some other extensions are not
      supported.  For example,
      - `eglMakeCurrent` is always called with `EGL_NO_SURFACE`.  It fails
      	without `EGL_KHR_surfaceless_context`.
- `wlr_gles2_renderer_create` creates `wlr_renderer`
  - these extensions are required
    - `GL_EXT_texture_format_BGRA8888`
    - `GL_EXT_unpack_subimage`

## output

- a compositor normally wraps `wlr_output` in a `wlr_output_damage`
  - when a `wlr_output` needs a frame, it calls `wlr_output_send_frame` to
    send a frame signal
  - `wlr_output_damage` handles the signal in `output_handle_frame` and sends
    its own frame signal
  - `damage_handle_frame` in sway handles the signal
    - `output_repaint_timer_handler` uses `wlr_renderer` to render the frame
      - `wlr_renderer_begin`
      - render
      - `wlr_renderer_end`
    - it calls `wlr_output_commit` after rendering
- output power on and off
  - `wlr_output_state_set_enabled` controls output power state
  - when output is off, `wlr_output` does not ask for frames

## vkms

- `WLR_BACKENDS=drm`
  - wlroots tries wayland backend, x11 backend, and then drm/libinput in order
  - this envvar explicitly requests drm backend
  - call sequence
    - `wlr_backend_autocreate`
- `LIBSEAT_BACKEND=noop`
  - libseat tries `seatd` and `logind` backends
  - this envvar explicitly requests noop backend
    - because the session is noop, there is no input nor drm device
  - call sequence
    - `wlr_backend_autocreate`
    - `session_create_and_wait`
    - `wlr_session_create`
    - `libseat_session_init`
    - `libseat_open_seat`
    - `noop_open_seat`
- `WLR_DRM_DEVICES=/dev/dri/by-path/platform-vkms-card`
  - this envvar explicitly adds drm devices
  - call sequence
    - `wlr_backend_autocreate`
    - `attempt_drm_backend`
    - `wlr_session_find_gpus`
- `WLR_RENDER_DRM_DEVICE` can explicitly specify the rendernode
  - it uses the primary node fd from the drm backend by default
  - call sequence
    - `wlr_renderer_autocreate`
- `WLR_RENDERER=pixman`
  - wlroots tries gles2, vulkan, and pixman backends in order
    - it tries pixman only when the drm device has no rendernode
  - the envvar explicitly requests pixman backend
  - call sequence
    - `wlr_renderer_autocreate`
    - `renderer_autocreate_with_drm_fd`
  - gles backend
    - `EGL_EXT_platform_device` fails because swrast has no
      `EGL_DRM_DEVICE_FILE_EXT`
    - `EGL_KHR_platform_gbm` is used instead
      - `gbm_create_device` tries these DRI drivers in order
        - `vkms_dri.so`
        - `zink_dri.so`
        - `kms_swrast_dri.so` (it picks this)
        - `swrast_dri.so`
      - `eglInitialize` gets the DRI driver name from gbm and loads
        `kms_swrast_dri.so` as well
    - because it is swrast, wlroots rejects it unless
      `WLR_RENDERER_ALLOW_SOFTWARE=1` is set
- `WLR_LIBINPUT_NO_DEVICES=1`
  - wlroots fails when there is no input device in the session
  - the envvar explicitly allows it
  - call sequence
    - `wlr_backend_start`
    - `multi_backend_start`
    - `wlr_backend_start`
    - `backend_start`
- bugs
  - libseat `noop_open_seat` does not initialize `initial_setup` to true
  - mesa `dri_swrast_kms_init_screen` does not initialize
    `has_reset_status_query` to true
