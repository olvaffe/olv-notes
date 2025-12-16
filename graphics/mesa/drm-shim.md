Mesa DRM Shim
=============

## Build and Use

- specify `-Dtools=drm-shim` to build
- `LD_PRELOAD` to use
- `DRM_SHIM_DEBUG=1` for debug messages
- backend-specifics
  - freedreno
    - `FD_GPU_ID=630` to override the simulated gpu

## Internals

- `git grep ^PUBLIC src/drm-shim/drm_shim.c` shows the intercepted functions
- `init_shim` is called when an intercepted function is called by the driver
  - it saves away funcion pointers to the real functions
  - `get_dri_render_node_minor` picks a `/dev/dri/renderD%d` as the faked
    node, which can be non-existent
  - it also prepares to intercept `/sys/dev/char/226:%d`
  - `drm_shim_device_init` initializes `shim_device` and allocates a 4GB memfd
    as the heap
  - `drm_shim_driver_init` provided by the backend is called
    - the backend's main job is to provide `driver_ioctls` to handle ioctls
- intercepted `fopen`
  - it overrides the contents of files, mainly sysfs ones
  - the backend calls `drm_shim_override_file` to pick the files to override
- intercepted `open`
  - if the path is the render node intercepted, it opens and returns `/dev/null` instead
  - `drm_shim_fd_register` remembers the fd and creates a `struct shim_fd`.
    The struct has a hash table of GEM handles.
- intercepted `ioctl`
  - if it is a fd registerd with `drm_shim_fd_register`, drm-shim handles the ioctl 
- when a bo is created,
  - `drm_shim_bo_init` assigns an offset in the memfd-backed heap
  - `drm_shim_bo_get_handle` assigns a gem handle to the bo and manages the bo
    in a hash table
- intercepted `mmap`
  - when a bo is mmaped, drm-shim maps an area in the memfd-backed heap
