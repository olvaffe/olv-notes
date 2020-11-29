Sommelier
=========

## Run

- From the `.gyp`,
  - `XWAYLAND_PATH=/opt/google/cros-containers/bin/Xwayland`
  - `XWAYLAND_GL_DRIVER_PATH=/opt/google/cros-containers/lib`
  - `XWAYLAND_SHM_DRIVER=virtwl`
  - `SHM_DRIVER=virtwl-dmabuf`
  - `VIRTWL_DEVICE=/dev/wl0`
  - `PEER_CMD_PREFIX=/opt/google/cros-containers/lib/ld-linux-x86-64.so.2 ...`
  - `FRAME_COLOR=#f2f2f2`
  - `DARK_FRAME_COLOR=#323639`
- From systemd unit files,
  - `SOMMELIER_SCALE=1.0`
  - `SOMMELIER_DRM_DEVICE=/dev/dri/renderD128`, if exists
  - master
    - `WAYLAND_DISPLAY_VAR=WAYLAND_DISPLAY`
    - `--socket=wayland-0`, under `/run/user/1000`
  - X11
    - `DISPLAY_VAR=DISPLAY`
    - `XCURSOR_SIZE_VAR=XCURSOR_SIZE`
    - `SOMMELIER_XFONT_PATH=...`
    - `SOMMELIER_GLAMOR=1`
    - `-X --x-display=0`
- There can be multiple master sommeliers.  Each master sommelier is a wayland
  server that does almost nothing.  Whenever a client connects to it, the
  master instance forks a peer instance to serve the client.  The peer
  instance relays the traffic between the client and the host server.  Each
  new client appears as a new client for the host server.
- There can be multiple X11 sommeliers.  Each X11 sommelier has a single
  client, Xwayalnd, connected to it.  It replays the traffic between Xwayland
  and the host server.  Xwayland appears as a client for the host server.
- 2D wayland clients
  - each 2D wayland client is served by a peer instance
  - the peer instance gets the frame data from the app via `wl_shm`
  - the peer instance also creates another SHM shared with the host compositor
  - when the client attaches, the peer instance attaches this other SHM to the
    host server
  - when the client commits, the peer instance memcpy()s from the app's SHM to
    the peer's SHM and commits to the host server
- 3D wayland clients
  - each 3D wayland client is served by a peer instance
  - the peer instance gets the frame data from the app via `wl_drm`
  - a prime fd is also a host buffer; the host compositor can see it directly
  - when the client attaches a buffer, the peer instance attaches the the
    buffer to the host compositor
  - when the client commits, the peer instance commits to the host server
- X11 clients
  - Xwayland renders with glamor and is a 3D wayland client served by a X11
    sommelier
  - it does not matter whether the X11 clients are 2D or 3D

## Buffers

- when the host supports `wl_shm`, sommelier advertises `wl_shm`
  - client binds `wl_shm` global
    - sommelier binds host `wl_shm` global or `zwp_linux_dmabuf_v1` global
      depending on the SHM driver
  - client calls `wl_shm_create_pool`
    - sommelier relays it to host (which will fail due to unknown fd) or
      remembers the shmem fd depending on the SHM driver
  - client calls `wl_shm_pool_create_buffer`,
    - sommelier relays it to host or mmap the shmem fd depending on the SHM
      driver
- when the host supports `zwp_linux_dmabuf_v1`, sommelier advertises `wl_drm`
  - client binds `wl_drm` global
    - sommelier binds host `zwp_linux_dmabuf_v1` global
  - client calls `wl_drm_create_prime_buffer`
    - sommelier calls host `zwp_linux_buffer_params_v1_create_immed` (using
      vfd support)
- when a client
  - calls `wl_surface_attach` on a `wl_shm` buffer, depending on the shm
    driver,
    - `SHM_DRIVER_NOOP`: never goes this far
    - `SHM_DRIVER_DMABUF`: a gbm bo is created and host
      `zwp_linux_buffer_params_v1_create_immed` is called
    - `SHM_DRIVER_VIRTWL`: a vfd is created and host `wl_shm_create_pool` and
      `wl_shm_pool_create_buffer` are called
    - `SHM_DRIVER_VIRTWL_DMABUF`: a vfd is created and host
      `wl_shm_create_pool` and `wl_shm_pool_create_buffer` are called.  The
      difference is the vfd points to a host gbm bo instead of a host shmem fd

## VirtWL

- open `/dev/wl0` to get a device fd
- Create a socketpair.  One endpoint is used by `wl_display_connect_to_fd`,
  for Sommelier to talk to the device like it is a `wl_display`.  The other
  endpoint massages the wayland protocol traffic and `VIRTWL_IOCTL_SEND` to
  the device.
- `VIRTWL_IOCTL_NEW` to get a context fd.  When the context fd is readable,
  `VIRTWL_IOCTL_RECV` to read the data, massage the data, and write the data
  into the socketpair.
- `VIRTWL_IOCTL_NEW_ALLOC` asks the host device to create a memfd, map it,
  inject the pages into the guest, and return the gpa to the guest kernel
  driver
- `VIRTWL_IOCTL_NEW_DMABUF` asks the host device to create a GEM BO, map it,
  inject the pages into the guest, and return the gpa to the guest kernel
  driver
- FDs
  - only ids can be sent
  - the kernel has a `id->vfd` table for all known vfds
  - the host can send an id and asks to create a new vfd
  - `VIRTWL_IOCTL_NEW_ALLOC` and `VIRTWL_IOCTL_NEW_DMABUF` creates a new vfd
    too
  - virtio-gpu dmabufs can be sent or received as "foreign" ids

## X11 Sommelier

- Communication with Xwayland
  - Sommelier creates a first `socketpair`.  One endpoint is passed to
    Xwayland with `WAYLAND_SOCKET=<fd>`.  Xwayland's call to
    `wl_display_connect(NULL)` will connect to this fd.
  - Sommelier creates a second `socketpair`.  One endpoint is passed to
    Xwayland with `-displayfd <fd>`.  Xwayland writes the display number to the
    fd when Xwayland is ready
  - Sommelier creates the last `socketpair`.  One endpoint is passed to
    Xwayland with `-wm <fd>`.  Xwayland adds the fd as an X client (which is the
    window manager)
