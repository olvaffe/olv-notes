Gallium Radeon SI
=================

## winsys

- `amdgpu_winsys_create` creates the winsys
  - call stack
    - `amdgpu_winsys_create`
    - `radeonsi_screen_create`
    - `pipe_radeonsi_create_screen`
    - `pipe_loader_drm_create_screen`
  - the function calls back to `radeonsi_screen_create_impl` to create the
    screen
- `amdgpu_screen_winsys`, `amdgpu_winsys`, and `amdgpu_device_handle`
  - `amdgpu_device_handle` wraps a `struct drm_device`
    - when `amdgpu_device_initialize` is called twice, and the fds refer to
      the same `struct drm_device`, the same `amdgpu_device_handle` is
      returned
  - `amdgpu_winsys` wraps a `amdgpu_device_handle`
    - `dev_tab` makes sure there is a 1:1 mapping
    - `amdgpu_winsys::fd` and `amdgpu_device_get_fd()` always refer to the
      same `struct drm_file`
  - `amdgpu_screen_winsys` wraps a `struct drm_file`
    - when `amdgpu_winsys_create` is called twoce, and the fds refer to the same `struct drm_file`,
      the same `amdgpu_screen_winsys` is returned
  - `are_file_descriptions_equal` returns true when two fds refer to the same
    `struct drm_file`
