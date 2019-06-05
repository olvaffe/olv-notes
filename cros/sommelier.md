# Sommelier

## Run

* From the `.gyp`,
  * `XWAYLAND_PATH=/opt/google/cros-containers/bin/Xwayland`
  * `XWAYLAND_GL_DRIVER_PATH=/opt/google/cros-containers/lib`
  * `XWAYLAND_SHM_DRIVER=virtwl`
  * `SHM_DRIVER=virtwl-dmabuf`
  * `VIRTWL_DEVICE=/dev/wl0`
  * `PEER_CMD_PREFIX=/opt/google/cros-containers/lib/ld-linux-x86-64.so.2 ...`
  * `FRAME_COLOR=#f2f2f2`
  * `DARK_FRAME_COLOR=#323639`
* From systemd unit files,
  * `SOMMELIER_SCALE=1.0`
  * `SOMMELIER_DRM_DEVICE=/dev/dri/renderD128`, if exists
  * master
    * `WAYLAND_DISPLAY_VAR=WAYLAND_DISPLAY`
    * `--socket=wayland-0`, under `/run/user/1000`
  * X11
    * `DISPLAY_VAR=DISPLAY`
    * `XCURSOR_SIZE_VAR=XCURSOR_SIZE`
    * `SOMMELIER_XFONT_PATH=...`
    * `SOMMELIER_GLAMOR=1`
    * `-X --x-display=0`

## VirtWL

* open `/dev/wl0` to get a device fd
* Create a socketpair.  One endpoint is used by `wl_display_connect_to_fd`,
  for Sommelier to talk to the device like it is a `wl_display`.  The other
  endpoint massages the wayland protocol traffic and `VIRTWL_IOCTL_SEND` to
  the device.
* `VIRTWL_IOCTL_NEW` to get a context fd.  When the context fd is readable,
  `VIRTWL_IOCTL_RECV` to read the data, massage the data, and write the data
  into the socketpair.

## X11 Sommelier

* Communication with Xwayland
  * Sommelier creates a first `socketpair`.  One endpoint is passed to
    Xwayland with `WAYLAND_SOCKET=<fd>`.  Xwayland's call to
    `wl_display_connect(NULL)` will connect to this fd.
  * Sommelier creates a second `socketpair`.  One endpoint is passed to
    Xwayland with `-displayfd <fd>`.  Xwayland writes the display number to the
    fd when Xwayland is ready
  * Sommelier creates the last `socketpair`.  One endpoint is passed to
    Xwayland with `-wm <fd>`.  Xwayland adds the fd as an X client (which is the
    window manager)
