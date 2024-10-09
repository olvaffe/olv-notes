wlroots
=======

## Initialization

- a compositor typically calls these functions during init
  - `wl_display_create` creates a `wl_display`
  - `wl_display_get_event_loop` returns the `wl_event_loop` from the display
  - `wlr_backend_autocreate` creates a `wlr_backend` and a `wlr_session` from
    the event loop
  - `wlr_renderer_autocreate` creates a `wlr_renderer` fom the backend
  - `wlr_allocator_autocreate` creates a `wlr_allocator` from the backend and
    the renderer
  - enables various wayland protocol and extensions
- `wlr_backend_autocreate`
  - there are multiple backends
    - `wayland` for sway-on-wayland
    - `x11` for sway-on-x11
    - `headless` for headless sway (useful for vnc-only)
    - `drm` and `libinput` for bare metal sway
  - `WLR_BACKENDS` can specify the comma-separated list of backends
  - otherwise,
    - if `WAYLAND_DISPLAY` or `WAYLAND_SOCKET` is set, use `wayland` backend
    - if `DISPLAY` is set, use `x11` backend
    - otherwise, use `drm` and `libinput` backend
      - they require a seat session from `libseat`
        - `libseat` in turns supports multiple backends
          - `seatd` talks to `seatd`
          - `logind` talks to `systemd-logind`
          - `builtin` talks to embedded `seatd`
          - `noop` is nop
        - `LIBSEAT_BACKEND` can specify the backend explicitly, otherwise it
          tries all backends in order
      - unless `WLR_LIBINPUT_NO_DEVICES` is set, `libinput` backend must exist
        and must enuemrate input devices
      - `drm` backend must enumerate drm primary nodes and it manages all of
        them by default
        - `WLR_DRM_DEVICES` can override
- `wlr_renderer_autocreate`
  - there are multiple renderers
    - `gles2` for egl/gles2
    - `vulkan` for vulkan
    - `pixman` for sw
  - `WLR_RENDERER` can specify the renderer explicitly
  - otherwise,
    - it tries `gles2`
      - it queries the primary drm fd from the backend, unless
        `WLR_RENDER_DRM_DEVICE` specifies a node explicitly
      - it inits EGL using
        - `EGL_PLATFORM_DEVICE_EXT` with the drm fd if supported, or
        - `EGL_PLATFORM_GBM_KHR` with the drm fd
    - it falls back to `pixman` if the backend drm is display-only
    - `vulkan` is still experimental and must be specified explicitly
      - it queries the drm fd from the backend similar to in `gles2`
      - it requires `VK_EXT_physical_device_drm` and uses the physical device
        matching the drm fd 
- `wlr_allocator_autocreate`
  - there are multiple allocators
    - gbm, using `gbm_bo_create`
    - shm, using `shm_open`
    - dumb, using `drmModeCreateDumbBuffer`
  - it queries the drm fd from the backend or the renderer
  - it picks the first allocator whose buffers can be presented by the backend
    - e.g., `drm` backend requires `WLR_BUFFER_CAP_DMABUF`
    - note that dumb allocator requires a primary drm fd

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

## seatd

- <https://git.sr.ht/~kennylevinsen/seatd>
  - the codebase is smaller
  - `/run/seatd.sock` expects the user to be in `seat` group
- initialization
  - `server_init` calls `seat_create` with `seat0` and `vt_bound` set
  - `open_socket` listens on `/run/seatd.sock`
  - `poller_poll` polls the socket
- `server_handle_connection` handles connections to the socket
  - `server_add_client` calls `client_create` to create a client
- `client_handle_connection` handles client opcodes
  - `CLIENT_OPEN_SEAT` is handled by `handle_open_seat`
    - `seat_add_client`
      - `seat_update_vt` sets `seat->cur_vt` and `client->session` to the
        current vt (usually 1)
    - `seat_open_client`
      - `vt_open` opens the current vt and switches to
        `VT_PROCESS`/`KD_GRAPHICS`/etc
  - `CLIENT_CLOSE_SEAT` is handled by `handle_close_seat`
    - `seat_remove_client` closes all opened devices
    - `seat_close_client` closes the current client (and activates the next
      client if any)
      - `vt_close` restores the vt
  - `CLIENT_OPEN_DEVICE` is handled by `handle_open_device`
    - `seat_open_device` opens a device node and adds it to `client->devices`
      - if `SEAT_DEVICE_TYPE_DRM`, the fd is made drm master
      - if `SEAT_DEVICE_TYPE_EVDEV`, no special handling
  - `CLIENT_CLOSE_DEVICE` is handled by `handle_close_device`
    - `seat_close_device` closes the device
  - `CLIENT_SWITCH_SESSION` is handled by `handle_switch_session`
    - because `vt_bound` is set, `vt_switch` initiates the switch 
  - `CLIENT_DISABLE_SEAT` is handled by `handle_disable_seat`
    - `seat_ack_disable_client` acks the disabling of the active client
  - `CLIENT_PING`
- session switch
  - user presses `CTRL-ALT-Fx`
  - compositor receives `XKB_KEY_XF86Switch_VT_x` and calls
    `libseat_switch_session` 
  - seatd receives `CLIENT_SWITCH_SESSION`
    - `vt_switch` switches vt
    - `server_handle_vt_rel` calls `seat_vt_release`
      - `seat_disable_client` disables the active client and sends
        `SERVER_DISABLE_SEAT` to the client
      - compositor calls `libseat_disable_seat` in response to
        `SERVER_DISABLE_SEAT`
        - seatd receives `CLIENT_DISABLE_SEAT`
      - `vt_ack` releases the vt
    - `server_handle_vt_acq` calls `seat_vt_activate`
      - `seat->cur_vt` is updated
      - `vt_ack` acquires the vk
      - `seat_activate` picks the next client for the new vt and calls
        `seat_open_client` to open the client and send `SERVER_ENABLE_SEAT`
- libseat logind backend
  - `seat_impl::open_seat`
    - it determines the session id which is from `XDG_SESSION_ID=c1`
    - `sd_session_get_seat` returns the seat name if the session is attached
      to a seat
    - `sd_bus_default_system` returns the system bus
    - `org.freedesktop.login1.Manager.GetSession` retruns
      `/org/freedesktop/login1/session/c1`
      - `dbus-send --system --dest=org.freedesktop.login1 --print-reply /org/freedesktop/login1 org.freedesktop.login1.Manager.GetSession string:c1`
    - `org.freedesktop.login1.Manager.GetSeat` retruns
      `/org/freedesktop/login1/seat/seat0`
    - add signal matches
      - `PauseDevice` and `ResumeDevice` of the session
      - `PropertiesChanged` of the session and the seat
    - `org.freedesktop.login1.Session.Activate` to activate the session
    - `org.freedesktop.login1.Session.TakeControl` to take control
    - `org.freedesktop.login1.Session.SetType` to set the type (wayland)
  - `seat_impl::disable_seat` is nop
  - `seat_impl::close_seat`
    - `org.freedesktop.login1.Session.ReleaseControl`
  - `seat_impl::seat_name` returns `sd_session_get_seat`
  - `seat_impl::open_device`
    - `org.freedesktop.login1.Session.TakeDevice`
  - `seat_impl::close_device`
    - `org.freedesktop.login1.Session.ReleaseDevice`
  - `seat_impl::switch_session`
    - `org.freedesktop.login1.Seat.SwitchTo`
  - `seat_impl::get_fd`
    - `sd_bus_get_fd`
  - `seat_impl::dispatch`
    - `sd_bus_process`

## Renderer

- init stack
  - `dri2_initialize_device`
  - `dri2_initialize`
  - `eglInitialize`
  - `egl_init_display`
  - `egl_init`
  - `wlr_egl_create_with_drm_fd`
  - `wlr_gles2_renderer_create_with_drm_fd`
  - `renderer_autocreate_with_drm_fd`
  - `wlr_renderer_autocreate`
  - `server_init`
  - `main`
- surface commit stack
  - `import_aux_info`
  - `iris_resource_finish_aux_import`
  - `iris_resource_from_handle`
  - `dri2_create_image_from_winsys`
  - `dri2_create_image_from_fd`
  - `dri2_from_dma_bufs2`
  - `dri2_create_image_dma_buf`
  - `dri2_create_image_khr`
  - `dri2_create_image`
  - `_eglCreateImageCommon`
  - `eglCreateImageKHR`
  - `wlr_egl_create_image_from_dmabuf`
  - `gles2_texture_from_dmabuf`
  - `gles2_texture_from_dmabuf_buffer`
  - `gles2_texture_from_buffer`
  - `wlr_texture_from_buffer`
  - `wlr_client_buffer_create`
  - `surface_apply_damage`
  - `surface_commit_state`
  - `surface_handle_commit`
