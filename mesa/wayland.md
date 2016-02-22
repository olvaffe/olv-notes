Wayland
=======

## window, the utility library

* Internals
  * window utility library is based on glib and cairo
* Initialization
  * invoke `display_create` to create a `struct display`
    * internally, it is a `struct wl_display`
    * after the connection to the server is made, `wl_display_iterate` is called
      to process all existing connection events before going on
    * when there is cairo-gl, a `cairo_device_t` is created
    * `display_create_surface_from_file` is called for each pointer images.  A
      GEM or SHM bo is allocated, and a `struct wl_buffer` is created for the
      bo.  And a `cairo_surface_t` is also created from the bo.
    * finally, shadow, active frame, and inactive frame cairo surfaces are
      created (for decoration)
  * invoke `window_create` to create a `struct window`
    * internally, it is a `struct wl_surface`
  * `window_set_decoration` disables window decoration
* Drawing
  * invoke `window_draw` to prepare the window and draw the decoration
    * when decoration is disabled, it only prepares the window by creating a bo,
      `struct wl_buffer`, and `cairo_surface_t`.
  * invoke `window_get_surface` to get the `cairo_surface_t` of the window
  * a `cairo_t` is created to draw the surface
  * flush and destry the surface
  * finally, invoke `window_flush`
    * the `struct wl_buffer` of the cairo surface is attached to the `struct
      wl_surface`.  The `struct wl_surface` is mapped (shown).
* Misc
  * `window_set_user_data` to associate user data with a window
  * `display_get_display` to return the `struct wl_display`
  * `window_set_child_size` and `window_get_child_rectangle` are used to specify
    the size of the window that can be used by the app, and to get the app
    usable area.
* Mainloop
  * invoke `display_run` to enter the mainloop

## Connection

* `struct wl_connection`
  * A connection is created with an fd and an update callback
    * update is called to indicate the connection is now readable or writable
  * Two internal ring buffers, in and out, are used for buffered I/O
    * `WL_CONNECTION_WRITABLE` means the `out` is not empty
    * `WL_CONNECTION_READABLE` is always set
  * Another two internal ring buffers, fds_in and fds_out, are used fd exchange
  * data in `in` can be copied out by `wl_connection_copy`; the tail in `in` is
    not incremented until `wl_connection_consume`
  * `out` is written to by `wl_connection_write`.  If `out` was empty,
    `WL_CONNECTION_WRITABLE` is set
  * `wl_connection_data` reads from and writes to fd
    * data read are buffered in `in`
    * fds read are buffered in `fds_in`
    * data in `out` and fds in `fds_out` are written
    * the size of the data in `in` are returned
* Signatures
  * `u` unsigned int
  * `i` signed int
  * `n` new object
  * `s` string length + string + extra `void *` (string pointer for ffi call)
  * `o` object id + extra `void *` (object pointer for ffi call)
  * `a` array length + array data + extra `void *` (array pointer for ffi call) + extra `struct wl_array`
  * `h` fd + extra `uint32_t` (marshalled directly to `fds_out`)
* Methods and Events
  * the same protocol format
  * an event is at least two `uint32_t` (header)
    * the first one is the object id
    * the second one is the event size (higher 16 bits) and opcode (lower 16
      bits).
  * after deciding the receiving object of the event and the opcode,
    `wl_connection_demarshal` is called to construct a closure
    * the args is the event signature plus 2 pointers: one for user data and the
      other for the object

## Wayland IPC

* Interfaces
  * the protocol defines interfaces
  * an interface is represented by a `struct wl_interface`
    * `name`/`version`: the name and version of the interface
    * `methods`/`events`:  methods/events of the interface.  Each of them is
      described by a `struct wl_message`, which consists of the name and the
      signature
  * everything must be described by an interface, including `struct wl_display` 
* Objects
  * an object is represented by a `struct wl_object`
  * an object consists of an interface, an implementation, and an id.
  * everything is an object, including `struct wl_display`.  A display has id 1.
* Proxies
  * a proxy is a data strucutre to make an interface appear to be a local object
  * listerners can be added to a proxy to handle events
  * methods can be invoked directly with a proxy

## Client API

* On client side, each object is also a proxy
* `wl_display_iterate` performs I/O to the server.  The read data are events and
  are dispatched to proxies.
* `wl_proxy_create_for_id` creates a proxy for an object id
  * a proxy for a server created object (display, compositor, ...)
* `wl_proxy_create` allocates an object id and creates a proxy for it
  * a proxy for a client created object (surface, buffer, drag, ...)
  * it is called a resource (of a client) on the server

## Server API

* `wl_display_create` creates a server display
  * a display has a `wl_event_loop`
  * also a hash table for objects
* `wl_display_add_socket` adds a listening socket
* `wl_client_create` is called whenever a client connects
  * when there are data to read, `wl_client_connection_data` is called.  It
    processes all methods (input)
  * a connection for I/O is created
  * a range event and a global event for each global are posted
* `wl_client_add_resource`
  * a client has a list of resources
  * they will be gone with the client
  * things like `wl_frame_listener` are also resources
* `wl_display_add_global` adds a global object
* `wl_client_post_event` sends an event to a client
* display interface
  * `sync` method causes a `sync` event to be posted immediately
  * `frame` method causes a `frame` event to be posted when the next frame is
    completed

## Server Repaint

* Transformation
  * `struct wl_matrix` is column-major, same as a GL matrix
  * `wlsc_matrix_multiply(M, N) = N x M`
  * surface transform: scale and then translate
    * from `[0, 1]*[0, 1]` to `[x, x+w]*[y, y+h]`
  * output transform: translate and then scale
    * from `[x, x+w]*[y, y+h]` to `[-1, 1]*[-1, 1]`
  * shader transform: surface transform and then output transform
* Y==0 is top
* whenever a repaint is required (surface damaged, mapped, or destroyed; input
  device moved), `wlsc_compositor_schedule_repaint` is called.  It schedules
  `repaint` to be called in 1ms.
* `repaint` draws `surface_list` in reverse order and draws `input_device_list`
  in normal order.  It then calls `present`
* `present` makes sure the new frame is visible on the screen.  It may schedule
  a page flip or do a copy from back to front.  When the new frame is finally
  visible, `wlsc_compositor_finish_frame` is called.  That sends a frame event
  to each interested client.

## Surface Interface

* `wl_surface_attach` attaches a buffer to a surface
  * a surface is like a texture object and a buffer is like a texture image
* `wl_surface_map` maps a surface onto the screen
  * the geometry needs not be the same as the buffer size.  A matrix will be
    applied
* `wl_surface_damage` marks a region of the attached buffer dirty
  * so that the texture may be updated and a repaint is scheduled
* `wl_display_sync_callback`

## DnD

* shell `create_drag` request
* `drag` requests
  * `offer`: add a mime type to the offer
  * `activate`: activate an offer
  * `destroy`: destroy the drag
* `drag` events
  * `target`: sent to the source for the accepted mime type
  * `finish`: sent to the source to start sending data
* `drag_offer` requests
  * `accept`: a target can accept an offer with the given mime type
  * `receive`: a target will receive the data from the given pipe
* `drag_offer` events
  * `offer`: sent to the target for the mime types supported
  * `pointer_focus`: sent to the new target
  * `motion`: sent to the target while dragging around
  * `drop`: sent to the target before starting receiving data
* In other words, the communications between source and target are
  * mouse down
  * src calls `shell::create_drag` to create a drag
  * src calls `drag::offer` to add supported mime types
  * src calls `drag::activate` to create a drag offer
  * target receives `drag_offer::offer` for the supported types
  * target receives `drag_offer::pointer_focus` or `drag_offer::motion`
  * target calls `drag_offer::accept` to accept the offer with the given mime
    type
  * src receives `drag::target` for the accepted mime type
    * src usually changes the pointer image here
  * mouse up
  * target receives `drag_offer::drop` to receive the offer
  * target calls `drag_offer::receive` to receive the data from the given pipe
  * src receives `drag::finish` to start sending the data
  * src calls `drag::destroy` to destroy the drag

## Input Devices

* `keyboard_focus` of an input device is the surface that will receive keyboard
  events of the device.  `keyboard_focus` is switched by clicking another
  surface.
* `pointer_focus` of an input device is the surface that will receive
  motion/button events of the device.  `pointer_focus` is usually switched by
  moving the pointer into another surface.  The exception is that when grab is
  active.
* grab types
  * `WLSC_DEVICE_GRAB_MOTION` the device is grabbed by the `pointer_focus`
    surface for motion events.  Pointer focus will not change even when the
    pointer goes outside the surface.
  * `WLSC_DEVICE_GRAB_MOVE` the device is grabbed by the `grab_surface` for
    moving the surface itself.  No motion/button events will be sent during the
    grab.  Instead, shell `configure` events are sent.
  * `WLSC_DEVICE_GRAB_RESIZE` the device is grabbed by the `grab_surface` for
    resizing the surface itself.  No motion/button events will be sent during
    the grab.  Instead, shell `configure` events are sent.
  * `WLSC_DEVICE_GRAB_DRAG` the device is grabbed by the `grab_surface` for
    dragging.  No motion/button events will be sent during the grab.  Instead,
    drag offer `motion`/`pointer_focus` events are sent.
* `notify_key` is called when a key is pressed.  It handles input events as well
  as does some WM works.
  * `Ctrl-Alt-Backspace` kills the server
  * an array of keys ever pressed is updated.  It is sent with `keyboard_focus`
    event so that a client can restore the keyboard state.
  * `key` event is then sent
* `notify_motion` is called when a mouse is moved.  It handles input events as
  well as does some WM works.
  * when there is no grab, `wlsc_input_device_set_pointer_focus` is called.  a
    `motion` event is sent.
* `notify_button` is called when a mouse button is clicked.  It handles input
  events as well as does some WM works.
  * it is ignored if there is no `pointer_focus`
  * if there is no grab, the focused surface is raised and grabs the input
    device.  `wlsc_input_device_set_keyboard_focus` is called to set
    `keyboard_forcus` to the surface.  An event of the same name is sent to the
    client losing the focus and the client gaining the focus
  * WM code kicks in to see if the button event triggers `shell_move` or
    `shell_resize`.  If neither, a `device_button` event is sent to the client.
  * `wlsc_input_device_end_grab` is called if the button is released.  It in
    turn calls `wlsc_input_device_set_pointer_focus`

overview
* (client) invokes wl_compositor_create_surface to create surface
* (client) invokes wl_surface_attach to attach a gem buffer to the surface
* (client) invokes wl_surface_map to specify the geometry of the surface
* (client) invokes wl_compositor_commit to commit changes
* (server) upon commit, compositor invokes schedule_repaint
* (server) upon map, surface remembers the new geometry
* (server) upon attach, surface creates a EGLSurface for the specified gem buffer
* (server) upon create_surface, compositor malloc()s a new surface
* after attach, the gem buffer belongs to the server
* further modifications should go through surface copy method

libwayland-server.so :				\
	wayland.o				\
	event-loop.o				\
	connection.o				\
	wayland-util.o				\
	wayland-protocol.o

idea
* everything is wl_object, including wl_display itself
* all objects are stored in display->objects.  They are either from display or
  clients (e.g., through create_surface).
* an wl_object consists of id, interface, and implementation.  display's
  objects has id in [0, 255).  0 means no such id, 1 is the id of the display.
  When a client connects, a range is reserved for the client.  When the range
  is running out, another range is reserved.
* an wl_inteface describes the events and methods of a wl_object.
  the details are stored in wl_message, which consists of name, signature, and types (unused?)

protocol
* wl_display_interface: events like invalid_object, invalid_method, no_memory, global, ...
* wl_compositor_interface: methods like create_surface and commit;
			   events like acknowledge and frame
* wl_surface_interface: methods like destroy, attach, map, copy, and damage.
* wl_input_device_interface: events like motion, button, keyboard_focus
* wl_output_interface: events like geometry

wl_connection
* a connection is established for each client.
* I/O to fd happens in wl_connection_data.  When WL_CONNECTION_READABLE,
  "in" buffer is updated.  data is read from fd, and in->head is incremented.
  When WL_CONNECTION_WRITABLE, "out" buffer is updated.  data is written to fd,
  and out->tail is incremented.  If "out" buffer is empty after writing,
  connection->update is called with WL_CONNECTION_READABLE.
* wl_connection_copy and wl_connection_consume copy data from "in" buffer.
* wl_connection_write copies data to "output" buffer.  If "out" buffer was
  empty before copy, connection->update is called with
  WL_CONNECTION_READABLE | WL_CONNECTION_WRITABLE.
* In wl_connection_create, connection->update is called with WL_CONNECTION_READABLE.
* protocol: SENDER[32]OPCODE[16]SIZE[16]DATA[variable]
* signature: o -> object id, n -> an unallocated object id
* wl_connection_vmarshal marshals the arguments and invokes wl_connection_write.
* wl_connection_demarshal demarshals data from wl_connection_copy and
  invokes supplied callback.

wl_event_loop
* There are four kinds of sources: fd, timer, signal, and idle.  Through the
  use of signalfd(2) and timerfd_create(2), signal and timer sources are
  treated like fd sources, which are epoll_wait()ed.
* wl_event_loop_wait invokes epoll_wait to wait for a max of 32 epoll_event.
  For each event, source->interface->dispatch is invoked.  If there is
  no event, dispatch_idles is invoked and all idle sources is invoked
  _and_ destroyed.

wayland display:
* init seq: wl_display_create, wl_display_set_compositor, wl_display_add_socket, and wl_display_run.
* when a new connection is made, a new client is created and client->source is
  added to display event loop.  WL_DISPLAY_RANGE and WL_DISPLAY_GLOBAL for
  each global are sent to the new client.  Then, each global is informed of
  the new client, which gives it a chance to send additional events.  Note
  that client is not stored in any of wl_display's lists.
* dispatch happens in wl_client_connection_data, which is invoked when data is available.
* When a client commits, compositor schedule_repaint and wl_client_send_acknowledge.
* When the compositor repaints, wl_display_post_frame is invoked and all clients on
  display->pending_frame_list is sent WL_COMPOSITOR_FRAME.  display->pending_frame_list is cleared.
* repaint exits early if not needed.  The seq is attach/map -> commit -> acknowledge -> repaint -> frame

wayland-system-compositor :			\
	wayland-system-compositor.o		\
	evdev.o					\
	cairo-util.o				\
	wayland-util.o

evdev_input_device:
* evdev_input_device_create takes a wlsc_input_device, and a display and path
* path is opened (for input events) and added to display event loop
* on events, notify_button, notify_key, or notify_motion is called upon wlsc_input_device

wayland-system-compositor.c
* a wlsc_compositor has a list of wlsc_output and wlsc_input_device
  respectively.  They are created through create_output and
  evdev_input_device_create.  ATM, they are called once.  That is, there is
  only one output and one input.
* a udev rule is installed to mark all mouse and kbd, and card0.  All mice and
  kbds report to the single wlsc_input_device.
* in create_output, init_egl is called to get EGLDisplay, EGLConfig, and
  EGLContext, which is stored in ec.  A wlsc_output is malloc()ed.  Current drm
  mode is queried and a gem buffer of size mode->hdisplay * mode->vdisplay * 4 is
  created, which is used as fb.  It then invokes eglCreateSurfaceForName to
  create a EGLSurface on the gem buffer.  The surface is stored in output->surface.
  Finally, background_create is invoked to create output->background.
* at the end of init_libudev, pointer_create is invoked to create pointer sprite.
* on notify_*, device->grab_surface or device->keyboard_focus might change.
  event is reported to the related surface.  compositor repaint is scheduled.


