# Weston

## libwindow, the utility library

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
    - the `struct wl_buffer` of the cairo surface is attached to the
      `struct wl_surface`.  The `struct wl_surface` is mapped (shown).
- Misc
  - `window_set_user_data` to associate user data with a window
  - `display_get_display` to return the `struct wl_display`
  - `window_set_child_size` and `window_get_child_rectangle` are used to specify
    the size of the window that can be used by the app, and to get the app
    usable area.
- Mainloop
  - invoke `display_run` to enter the mainloop

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
