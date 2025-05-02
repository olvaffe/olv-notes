Wayland
=======

## Wayland Repos

- <https://gitlab.freedesktop.org/wayland/wayland.git>
  - core wayland code and protocol
  - see `Wayland Core` below for the core code
  - the core protocol is defined in `wayland.xml`
    - `<protocol name="wayland">`
    - `<interface name="wl_display" version="1">`
    - `<interface name="wl_registry" version="1">`
    - `<interface name="wl_callback" version="1">`
    - `<interface name="wl_compositor" version="6">`
    - `<interface name="wl_shm_pool" version="2">`
    - `<interface name="wl_shm" version="2">`
    - `<interface name="wl_buffer" version="1">`
    - `<interface name="wl_data_offer" version="3">`
    - `<interface name="wl_data_source" version="3">`
    - `<interface name="wl_data_device" version="3">`
    - `<interface name="wl_data_device_manager" version="3">`
    - `<interface name="wl_shell" version="1">`
      - deprecated by `xdg_shell`
    - `<interface name="wl_shell_surface" version="1">`
    - `<interface name="wl_surface" version="6">`
    - `<interface name="wl_seat" version="10">`
    - `<interface name="wl_pointer" version="10">`
    - `<interface name="wl_keyboard" version="10">`
    - `<interface name="wl_touch" version="10">`
    - `<interface name="wl_output" version="4">`
    - `<interface name="wl_region" version="1">`
    - `<interface name="wl_subcompositor" version="1">`
    - `<interface name="wl_subsurface" version="1">`
    - `<interface name="wl_fixes" version="1">`
- <https://gitlab.freedesktop.org/wayland/wayland-utils.git>
  - the only utility is `wayland-info`
- <https://gitlab.freedesktop.org/wayland/weston.git>
  - `weston` is a basic compositor and `libweston` can be used to build
    full-featured compositor
- <https://gitlab.freedesktop.org/wayland/wayland-protocols.git>
  - protocol phases
    - development phase
      - the protocol may be added to `experimental/`, and is actively
        developed in the form of an MR
      - MRs for clients and compositors are developed as test vehicles
      - incompatible changes are allowed
    - testing phase
      - the protocol is added to `staging/`
      - client and compositor implementations are encouraged
      - incompatible changes are NOT allowed
      - if design flaws are found, the procotol may see a major version bump
        or be entirely replaced
    - stable phase
      - the protocol is added to `stable/`
    - unstable phase
      - no longer used
      - the protocol is added to `unstable/`
      - interface names are prefixed by z
      - promotion to stable phase is always an incompatible change
  - directory tree
    - `stable` is for current stable protocols
    - `staging` is for protocols in the testing phase
    - `deprecated` is for deprecated protocols
    - `unstable` is historical and should not be used for new protocols
  - naming conventions
    - xml file path should be `foo/foo-vN.xml` for protocol `foo` at major
      version `N`
    - protocol and interface names
      - window-managing protocols should use `xdg_` prefix
      - fundamental protocols should use `wp_` prefix
        - os-specific ones should use `wp_<os>_` prefix
      - other protocols should use `ext_` prefix
      - all protocols should use `_vN` suffix, where N is the major version
  - stable protocols
    - `<protocol name="linux_dmabuf_v1">`
      - `<interface name="zwp_linux_dmabuf_v1" version="5">`
      - `<interface name="zwp_linux_buffer_params_v1" version="5">`
      - `<interface name="zwp_linux_dmabuf_feedback_v1" version="5">`
    - `<protocol name="presentation_time">`
      - `<interface name="wp_presentation" version="2">`
      - `<interface name="wp_presentation_feedback" version="2">`
    - `<<protocol name="tablet_v2">`
      - `<interface name="zwp_tablet_manager_v2" version="1">`
      - `<interface name="zwp_tablet_seat_v2" version="1">`
      - `<interface name="zwp_tablet_tool_v2" version="1">`
      - `<interface name="zwp_tablet_v2" version="1">`
      - `<interface name="zwp_tablet_pad_ring_v2" version="1">`
      - `<interface name="zwp_tablet_pad_strip_v2" version="1">`
      - `<interface name="zwp_tablet_pad_group_v2" version="1">`
      - `<interface name="zwp_tablet_pad_v2" version="1">`
    - `<protocol name="viewporter">`
      - `<interface name="wp_viewporter" version="1">`
      - `<interface name="wp_viewport" version="1">`
    - `<protocol name="xdg_shell">`
      - `<interface name="xdg_wm_base" version="6">`
      - `<interface name="xdg_positioner" version="6">`
      - `<interface name="xdg_surface" version="6">`
      - `<interface name="xdg_toplevel" version="6">`
      - `<interface name="xdg_popup" version="6">`
  - selected staging protocols
    - `<protocol name="alpha_modifier_v1">`
      - this allows a client to speicify the alpha for a surface
      - `<interface name="wp_alpha_modifier_v1" version="1">`
      - `<interface name="wp_alpha_modifier_surface_v1" version="1">`
    - `<protocol name="commit_timing_v1">`
      - it allows a client to specify a timestamp for presentation
      - `<interface name="wp_commit_timing_manager_v1" version="1">`
      - `<interface name="wp_commit_timer_v1" version="1">`
    - `<protocol name="drm_lease_v1">`
      - it allows a client to lease a drm device (and act as a vr compositor)
      - `<interface name="wp_drm_lease_device_v1" version="1">`
      - `<interface name="wp_drm_lease_connector_v1" version="1">`
      - `<interface name="wp_drm_lease_request_v1" version="1">`
      - `<interface name="wp_drm_lease_v1" version="1">`
    - `<protocol name="ext_data_control_v1">`
      - it allows a special client to act as a clipboard manager
      - `<interface name="ext_data_control_manager_v1" version="1">`
      - `<interface name="ext_data_control_device_v1" version="1">`
      - `<interface name="ext_data_control_source_v1" version="1">`
      - `<interface name="ext_data_control_offer_v1" version="1">`
    - `<protocol name="ext_image_capture_source_v1">`
      - it allows a client to select a capture source
      - `<interface name="ext_image_capture_source_v1" version="1">`
      - `<interface name="ext_output_image_capture_source_manager_v1" version="1">`
      - `<interface name="ext_foreign_toplevel_image_capture_source_manager_v1" version="1">`
    - `<protocol name="ext_image_copy_capture_v1">`
      - it allows a client to copy the contents of a capture source to a
        buffer
      - `<interface name="ext_image_copy_capture_manager_v1" version="1">`
      - `<interface name="ext_image_copy_capture_session_v1" version="1">`
      - `<interface name="ext_image_copy_capture_frame_v1" version="1">`
      - `<interface name="ext_image_copy_capture_cursor_session_v1" version="1">`
    - `<protocol name="fifo_v1">`
      - it allows a client to present multiple frames in fifo mode rather than
        mailbox mode
      - `<interface name="wp_fifo_manager_v1" version="1">`
      - `<interface name="wp_fifo_v1" version="1">`
    - `<protocol name="fractional_scale_v1">`
      - it allows a client to receive fractional scale
      - `<interface name="wp_fractional_scale_manager_v1" version="1">`
      - `<interface name="wp_fractional_scale_v1" version="1">`
    - `<protocol name="linux_drm_syncobj_v1">`
      - it allows a client to associate a timeline syncobj with a surface for
        explicit sync
      - `<interface name="wp_linux_drm_syncobj_manager_v1" version="1">`
      - `<interface name="wp_linux_drm_syncobj_timeline_v1" version="1">`
      - `<interface name="wp_linux_drm_syncobj_surface_v1" version="1">`
    - `<protocol name="tearing_control_v1">`
      - it allows a client to present async (disable vsync), for lower latency
        at the cost of tearing
      - `<interface name="wp_tearing_control_manager_v1" version="1">`
      - `<interface name="wp_tearing_control_v1" version="1">`
    - `<protocol name="xwayland_shell_v1">`
      - it is used by xwayland server and is hidden from regular clients
      - `<interface name="xwayland_shell_v1" version="1">`
      - `<interface name="xwayland_surface_v1" version="1">`
  - selected unstable protocols
    - `<protocol name="idle_inhibit_unstable_v1">`
      - it allows a client to inhibit screen saver/blanking
      - `<interface name="zwp_idle_inhibit_manager_v1" version="1">`
      - `<interface name="zwp_idle_inhibitor_v1" version="1">`
    - `<protocol name="input_method_unstable_v1">`
      - it is used by special clients such as input method daemons or virtual
        keyboards
        - I guess the input method daemon can grab the hw keyboard, process hw
          keys, and send utf8 text to the compositor
        - the compositor uses another protocol to send the utf8 text to app
          clients
      - `<interface name="zwp_input_method_context_v1" version="1">`
      - `<interface name="zwp_input_method_v1" version="1">`
      - `<interface name="zwp_input_panel_v1" version="1">`
      - `<interface name="zwp_input_panel_surface_v1" version="1">`
    - `<protocol name="text_input_unstable_v3">`
      - it is used by normal clients to receive utf8 text as input
      - `<interface name="zwp_text_input_v3" version="1">`
      - `<interface name="zwp_text_input_manager_v3" version="1">`

## Wayland core

- Binaries
  - `wayland-scanner` scans an Wayland protocol XML (`<name>.xml`) and outputs
    C code (`<name>-protocol.c`), client header (`<name>-client-protocol.h`),
    and server header (`<name>-server-protocol.h`).
  - `libwayland-server` is the core library for implementing a Wayland server
    - there is hand-written code whose header is `wayland-server-core.h`
    - there is scanner-generated code whose header is
      `wayland-server-protocol.h`
    - there is a copy of the util code whose header is `wayland-util.h`
  - `libwayland-client` is the core library for implementing a Wayland client
    - there is hand-written code whose header is `wayland-client-core.h`
    - there is scanner-generated code whose header is
      `wayland-client-protocol.h`
    - there is a copy of the util code whose header is `wayland-util.h`
  - `libwayland-cursor` is a helper to find and load Xcursor cursor images
    - only hand-written code whose header is `wayland-cursor.h`
  - `libwayland-egl` is a helper to create a `EGLNativeWindowType` for use
    with EGL
    - only hand-written code whose header is `wayland-egl.h`
    - there is also `wayland-egl-backend.h` used by EGL implementations to
      peek inside `wl_egl_window` and add hooks
- `libwayland-client`
  - basic
    - `wl_display` is the main object to connect to the server.  Internally, the
      connection is a `wl_connection` that wraps a socket fd with buffered io.
    - `wl_display` is also the proxy for a singleton object in the server
      exporting `wl_display` innterface
  - incoming event
    - Each incoming event targets a proxy.  It is read by the client from the
      socket fd, demarshalled to a `wl_closure`, and added to the
      `wl_event_queue` of the proxy
    - the client later dispatches closers in an event queue.  This invokes the
      corresponding method of the implementation for each closure
  - outgoing event
    - each outgoing event is generated by invoking a function of a proxy.  The
      regular function call is marshalled into a closure and written to the
      socket fd.
- `libwayland-server`
  - wayland protocols define many interfaces
  - a server supporting an interface can advertise it using a global
  - multiple server-side objects supporting the interface can be created

## `wl_connection`

- `wl_connection` is an abstraction of the socket fd to the compositor
  - there are `in`/`out` and `fds_in`/`fds_out` buffers
  - calling `wl_connection_data` with `WL_CONNECTION_WRITABLE` writes `out` and
    `fds_out` buffers to the socket.  If all bytes are written, the `update`
    callback is called with `WL_CONNECTION_READABLE`, meaning that there is
    nothing in the connection to be written (because `WL_CONNECTION_WRITABLE` is
    no longer set).  It returns the number of bytes in the `in` buffer.
  - calling `wl_connection_data` with `WL_CONNECTION_READABLE` reads data from
    the socket and stores them in `in` and `fds_in` buffers.
  - In `wl_connection_write`, data are put on the `out` buffer.
    `wl_connection_data` is called only when the buffer is full.  If the
    connection was empty, the `update` callback is called with both
    `WL_CONNECTION_WRITABLE` and `WL_CONNECTION_READABLE` set, meaning there are
    data to be written and read.
- `struct wl_connection`
  - A connection is created with an fd and an update callback
    - update is called to indicate the connection is now readable or writable
  - Two internal ring buffers, in and out, are used for buffered I/O
    - `WL_CONNECTION_WRITABLE` means the `out` is not empty
    - `WL_CONNECTION_READABLE` is always set
  - Another two internal ring buffers, fds_in and fds_out, are used fd exchange
  - data in `in` can be copied out by `wl_connection_copy`; the tail in `in` is
    not incremented until `wl_connection_consume`
  - `out` is written to by `wl_connection_write`.  If `out` was empty,
    `WL_CONNECTION_WRITABLE` is set
  - `wl_connection_data` reads from and writes to fd
    - data read are buffered in `in`
    - fds read are buffered in `fds_in`
    - data in `out` and fds in `fds_out` are written
    - the size of the data in `in` are returned
- Signatures
  - `u` unsigned int
  - `i` signed int
  - `n` new object
  - `s` string length + string + extra `void *` (string pointer for ffi call)
  - `o` object id + extra `void *` (object pointer for ffi call)
  - `a` array length + array data + extra `void *` (array pointer for ffi call) + extra `struct wl_array`
  - `h` fd + extra `uint32_t` (marshalled directly to `fds_out`)
- Methods and Events
  - the same protocol format
  - an event is at least two `uint32_t` (header)
    - the first one is the object id
    - the second one is the event size (higher 16 bits) and opcode (lower 16
      bits).
  - after deciding the receiving object of the event and the opcode,
    `wl_connection_demarshal` is called to construct a closure
    - the args is the event signature plus 2 pointers: one for user data and the
      other for the object

## Wayland IPC

- Interfaces
  - the protocol defines interfaces
  - an interface is represented by a `struct wl_interface`
    - `name`/`version`: the name and version of the interface
    - `methods`/`events`:  methods/events of the interface.  Each of them is
      described by a `struct wl_message`, which consists of the name and the
      signature
  - everything must be described by an interface, including `struct wl_display` 
- Objects
  - an object is represented by a `struct wl_object`
  - an object consists of an interface, an implementation, and an id.
  - everything is an object, including `struct wl_display`.  A display has id 1.
- Proxies
  - a proxy is a data strucutre to make an interface appear to be a local object
  - listerners can be added to a proxy to handle events
  - methods can be invoked directly with a proxy

## Client API

- On client side, each object is also a proxy
- `wl_display_iterate` performs I/O to the server.  The read data are events and
  are dispatched to proxies.
- `wl_proxy_create_for_id` creates a proxy for an object id
  - a proxy for a server created object (display, compositor, ...)
- `wl_proxy_create` allocates an object id and creates a proxy for it
  - a proxy for a client created object (surface, buffer, drag, ...)
  - it is called a resource (of a client) on the server

## Server API

- `wl_display_create` creates a server display
  - a display has a `wl_event_loop`
  - also a hash table for objects
- `wl_display_add_socket` adds a listening socket
- `wl_client_create` is called whenever a client connects
  - when there are data to read, `wl_client_connection_data` is called.  It
    processes all methods (input)
  - a connection for I/O is created
  - a range event and a global event for each global are posted
- `wl_client_add_resource`
  - a client has a list of resources
  - they will be gone with the client
  - things like `wl_frame_listener` are also resources
- `wl_display_add_global` adds a global object
- `wl_client_post_event` sends an event to a client
- display interface
  - `sync` method causes a `sync` event to be posted immediately
  - `frame` method causes a `frame` event to be posted when the next frame is
    completed
- `wl_event_loop`
  - There are four kinds of sources: fd, timer, signal, and idle.  Through the
    use of signalfd(2) and timerfd_create(2), signal and timer sources are
    treated like fd sources, which are epoll_wait()ed.
  - `wl_event_loop_wait` invokes `epoll_wait` to wait for a max of 32
    epoll_event.
    - For each event, `source->interface->dispatch` is invoked
    - If there is no event, `dispatch_idles` is invoked and all idle sources is
      invoked _and_ destroyed.

## How does a client work

- Connect to display
  - `wl_display_connect(name)` connects to the named socket, or `wayland-0` if
    `NULL` is given.  `wl_display` is returned.  Internally,
    - `wl_connection_create(fd, ...)` is called to create a `wl_connection` for
      the socket fd
  - `wl_display_add_global_listener` add a callback to be invoked on each global
    objects
  - `wl_display_iterate` is called to read inbound data
- Render with SHM
  - `display = wl_display_connect(NULL)` to connecto to the server and return
    `wl_display`
  - Listen for `wl_compositor` global and create `wl_compositor`
  - `surface = wl_compositor_create_surface(compositor)` to create `wl_surface`
  - an SHM buffer is created and an SHM-based `wl_buffer` is created with
    `buf = wl_shm_create_buffer(shm, fd, w, h, stride, vis)`
  - the SHM buffer is mapped for direct rendering
  - `wl_buffer_damage` is called to notify the buffer contents need revalidation
  - `wl_surface_attach` to attach the buffer to the surface
  - `wl_surface_damage` to notify the surface needs revalidation
- Render with EGL
  - `egldpy = eglGetDisplay(display)` to return the `EGLDisplay` of the
    `wl_display`
  - `native = wl_egl_window_create(surface)` to create the EGL native window
    from the `wl_surface`
  - `egl_surface = eglCreateWindowSurface(...)` to create the `EGLSurface` from
    the EGL native window
  - `eglMakeCurrent` and start rendering
  - after rendering a frame, `eglSwapBuffers` is called to present
- Render to pixmap (not used anywhere)
  - similar to rendering with SHM, except
  - A native pixmap is created instead of a SHM buffer
    - `pix = wl_egl_pixmap_create(w, h, vis, 0)`
  - `egl_surface = eglCreatePixmapSurface(...)` is called so that GL instead of
    CPU can be used for rendering 
    - `eglMakeCurrent` should work
    - `img = eglCreateImageKHR(..., EGL_NATIVE_PIXMAP_KHR, pix, NULL)`,
      `glEGLImageTargetTexture2DOES`, and render-to-texture should work too,
      without context switching
  - `wl_egl_pixmap_create_buffer` is called instead of `wl_shm_create_buffer`
    for presenting
- Render with cairo
  - the key is how the `cairo_surface_t` is created
    - SHM buffer and `cairo_image_surface_create_for_data` for CPU rendering
    - Pixmap and `cairo_gl_surface_create_for_texture` for GPU rendering
      - require `EGL_KHR_image_pixmap`
    - `wl_surface` and `cairo_gl_surface_create_for_egl` for GPU rendering
- More hints
  - after a buffer is attached to a surface, the buffer is owned by the
    compositor until the `release` event is received
  - if the compositor never sends the `release` event, clients will allocate a
    new buffer for each frame
  - a buffer may be attached to multiple surfaces
    - how does the `release` event work?

## How does a compositor work

- `wl_surface`
  - a surface has a texture object
  - when a buffer attached, the data are downloaded to the texture object
  - when a buffer is damaged, its data are re-downloaded
  - for DRM-based buffer, there is zero copy.
  - DRM-based buffers are not supported by core, but by mesa.  It is an internal
    mechanism
- init seq
  - `wl_display_create`
  - `wl_display_set_compositor`
  - `wl_display_add_socket`
  - `wl_display_run`
- when a new connection is made, a new client is created and `client->source`
  is added to display event loop
  - `WL_DISPLAY_RANGE` and `WL_DISPLAY_GLOBAL` for each global are sent to the
    new client
  - Then, each global is informed of the new client, which gives it a chance
    to send additional events.  Note that client is not stored in any of
    `wl_display`'s lists.
- dispatch happens in `wl_client_connection_data`, which is invoked when data
  is available
- When a client commits, compositor `schedule_repaint` and
  `wl_client_send_acknowledge`
- When the compositor repaints, `wl_display_post_frame` is invoked
  - all clients on `display->pending_frame_list` is sent
    `WL_COMPOSITOR_FRAME`
  - `display->pending_frame_list` is cleared
  - repaint exits early if not needed
- The seq is attach/map -> commit -> acknowledge -> repaint -> frame

## `wl_surface` and `wl_buffer`

- a `wl_surface` is a rectangular area that receives inputs and presents
  `wl_buffer`
  - many surface states are double-buffered;
  - protocol requests change the pending states
  - `wl_surface_commit` atomically makes all pending states current
    - the values of the pending states after commit are case by case
      - pending damage state becomes empty
      - pending scale state is unchanged
  - `wl_surface_attach` attaches a `wl_buffer` to a `wl_surface`
    - a surface is like a texture object and a buffer is like a texture image
  - `wl_surface_map` maps a surface onto the screen
    - the geometry needs not be the same as the buffer size.  A matrix will be
      applied
  - `wl_surface_damage` marks a region of the attached buffer dirty
    - so that the texture may be updated and a repaint is scheduled
  - `wl_surface_frame` returns a `wl_callback` that is used to notify frame
    presented
  - `wl_display_sync_callback`
- a `wl_buffer` is a content-provider for a `wl_surface`.  It is created from
  factory interfaces and can be attached to surfaces.
  - `release` event is used to notify that the buffer is released by the
    server and can be reused for other purposes (e.g., draw next frame)
- `wl_shm`
  - client allocates a shmem
    - client can access any area of the shmem
  - `wl_shm_create_pool` creates a `wl_shm_pool` from the shmem fd
    - server is granted access any area of the shmem
  - client suballocates from the shmem and call `wl_shm_pool_create_buffer`
    - client and server both know which area is used as the storage of the
      `wl_buffer`
- `wl_drm`
  - interface defined by Mesa, not Wayland
  - client allocates a GPU BO and export dmabuf
    - client can render to BO
  - `wl_drm_create_prime_buffer` to create a buffer from the dmabuf
    - server can sample from BO
  - originally, flink is used instead of dmabuf
- `zwp_linux_dmabuf_v1`
  - it is similar to `wl_drm` but with multi-planar and modifier support

## EGL

- `libwayland-drm` defines `wl_drm` interface that is for internal use
  - when a client connects, `wl_drm` advertises the DRM device name
  - `authenticate` can be used to auth a DRM fd
  - `create_buffer` can be used to create a `wl_buffer` for the named bo
- `EGL_WL_bind_wayland_display` is used by EGL-based Wayland servers to run on
  an alternative window system.
  - It allows a server `wl_display` to be bound to an `EGLDisplay` for the
    alternative window system.
  - It allows an `EGLImageKHR` to be created from a `wl_buffer`
  - It adds `wl_drm` interface to the `wl_display`.
    - DRM device name is from the alternative window system
    - `authenticate` is implemented using the alternative window system
    - `create_buffer` creates a `__DRIimage` from the named bo
- Client EGL support
  - The color buffers of an `EGLSurface` are allocated by the DRI driver.
  - For each color buffer, there is an associated `wl_buffer` created using
    `wl_drm` interface

## Xwayland

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

## weston libwindow, the utility library

- Internals
  - window utility library is based on glib and cairo
- Initialization
  - invoke `display_create` to create a `struct display`
    - internally, it is a `struct wl_display`
    - after the connection to the server is made, `wl_display_iterate` is called
      to process all existing connection events before going on
    - when there is cairo-gl, a `cairo_device_t` is created
    - `display_create_surface_from_file` is called for each pointer images.  A
      GEM or SHM bo is allocated, and a `struct wl_buffer` is created for the
      bo.  And a `cairo_surface_t` is also created from the bo.
    - finally, shadow, active frame, and inactive frame cairo surfaces are
      created (for decoration)
  - invoke `window_create` to create a `struct window`
    - internally, it is a `struct wl_surface`
  - `window_set_decoration` disables window decoration
- Drawing
  - invoke `window_draw` to prepare the window and draw the decoration
    - when decoration is disabled, it only prepares the window by creating a bo,
      `struct wl_buffer`, and `cairo_surface_t`.
  - invoke `window_get_surface` to get the `cairo_surface_t` of the window
  - a `cairo_t` is created to draw the surface
  - flush and destry the surface
  - finally, invoke `window_flush`
    - the `struct wl_buffer` of the cairo surface is attached to the `struct
      wl_surface`.  The `struct wl_surface` is mapped (shown).
- Misc
  - `window_set_user_data` to associate user data with a window
  - `display_get_display` to return the `struct wl_display`
  - `window_set_child_size` and `window_get_child_rectangle` are used to specify
    the size of the window that can be used by the app, and to get the app
    usable area.
- Mainloop
  - invoke `display_run` to enter the mainloop

## DnD

- shell `create_drag` request
- `drag` requests
  - `offer`: add a mime type to the offer
  - `activate`: activate an offer
  - `destroy`: destroy the drag
- `drag` events
  - `target`: sent to the source for the accepted mime type
  - `finish`: sent to the source to start sending data
- `drag_offer` requests
  - `accept`: a target can accept an offer with the given mime type
  - `receive`: a target will receive the data from the given pipe
- `drag_offer` events
  - `offer`: sent to the target for the mime types supported
  - `pointer_focus`: sent to the new target
  - `motion`: sent to the target while dragging around
  - `drop`: sent to the target before starting receiving data
- In other words, the communications between source and target are
  - mouse down
  - src calls `shell::create_drag` to create a drag
  - src calls `drag::offer` to add supported mime types
  - src calls `drag::activate` to create a drag offer
  - target receives `drag_offer::offer` for the supported types
  - target receives `drag_offer::pointer_focus` or `drag_offer::motion`
  - target calls `drag_offer::accept` to accept the offer with the given mime
    type
  - src receives `drag::target` for the accepted mime type
    - src usually changes the pointer image here
  - mouse up
  - target receives `drag_offer::drop` to receive the offer
  - target calls `drag_offer::receive` to receive the data from the given pipe
  - src receives `drag::finish` to start sending the data
  - src calls `drag::destroy` to destroy the drag

## Input Devices

- `keyboard_focus` of an input device is the surface that will receive keyboard
  events of the device.  `keyboard_focus` is switched by clicking another
  surface.
- `pointer_focus` of an input device is the surface that will receive
  motion/button events of the device.  `pointer_focus` is usually switched by
  moving the pointer into another surface.  The exception is that when grab is
  active.
- grab types
  - `WLSC_DEVICE_GRAB_MOTION` the device is grabbed by the `pointer_focus`
    surface for motion events.  Pointer focus will not change even when the
    pointer goes outside the surface.
  - `WLSC_DEVICE_GRAB_MOVE` the device is grabbed by the `grab_surface` for
    moving the surface itself.  No motion/button events will be sent during the
    grab.  Instead, shell `configure` events are sent.
  - `WLSC_DEVICE_GRAB_RESIZE` the device is grabbed by the `grab_surface` for
    resizing the surface itself.  No motion/button events will be sent during
    the grab.  Instead, shell `configure` events are sent.
  - `WLSC_DEVICE_GRAB_DRAG` the device is grabbed by the `grab_surface` for
    dragging.  No motion/button events will be sent during the grab.  Instead,
    drag offer `motion`/`pointer_focus` events are sent.
- `notify_key` is called when a key is pressed.  It handles input events as well
  as does some WM works.
  - `Ctrl-Alt-Backspace` kills the server
  - an array of keys ever pressed is updated.  It is sent with `keyboard_focus`
    event so that a client can restore the keyboard state.
  - `key` event is then sent
- `notify_motion` is called when a mouse is moved.  It handles input events as
  well as does some WM works.
  - when there is no grab, `wlsc_input_device_set_pointer_focus` is called.  a
    `motion` event is sent.
- `notify_button` is called when a mouse button is clicked.  It handles input
  events as well as does some WM works.
  - it is ignored if there is no `pointer_focus`
  - if there is no grab, the focused surface is raised and grabs the input
    device.  `wlsc_input_device_set_keyboard_focus` is called to set
    `keyboard_forcus` to the surface.  An event of the same name is sent to the
    client losing the focus and the client gaining the focus
  - WM code kicks in to see if the button event triggers `shell_move` or
    `shell_resize`.  If neither, a `device_button` event is sent to the client.
  - `wlsc_input_device_end_grab` is called if the button is released.  It in
    turn calls `wlsc_input_device_set_pointer_focus`

## `linux_dmabuf_v1`

- `zwp_linux_dmabuf_v1`
  - `create_params` creates a temporary `zwp_linux_buffer_params_v1` used to
    specify buffer parameters
  - `get_default_feedback` returns the global `zwp_linux_dmabuf_feedback_v1`
  - `get_surface_feedback` returns a surface `zwp_linux_dmabuf_feedback_v1`
- `zwp_linux_buffer_params_v1`
  - `add` adds a memory plane to the params
  - `create_immed` creates a `wl_buffer` based on the specified params
  - `create` creates a `wl_buffer` asynchronously
    - `created` returns the `wl_buffer`
    - `failed` indicates failure
- `zwp_linux_dmabuf_feedback_v1`
  - `format_table` sends a shmem to the client
    - the contents are an array of 16-byte element, where each element is
      - 4-byte format
      - 4-byte padding
      - 8-bite modifier
    - the contents must not mutate
    - to update the contents, a new shmem must be sent
  - `main_device` sends a `dev_t` to the client
    - this is the drm device that the compositor uses when falling back to GPU
      composition
  - compositor sends one or more tranches to the client, in descending order
    of preference, where each tranche is
    - `tranche_target_device` sends a `dev_t` to the client
      - this is the drm device that the compositor prefers to use to access
        buffers belonging to this tranche
    - `tranche_flags` sends a `flags` to the client
      - `scanout` means the compositor will attempt to do direct scanout first
        before falling back to gpu composition
    - `tranche_formats` sends an array of u16 to the client
      - they are indices into the format table
    - `tranche_done` marks the end of the current tranche
  - `done` marks the end of all events
