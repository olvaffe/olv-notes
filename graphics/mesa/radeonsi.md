Gallium Radeon SI
=================

## `GL_RENDERER`

- e.g., `AMD Radeon Graphics (renoir, LLVM 16.0.6, DRM 3.54, 6.5.4)`
- `si_init_renderer_string`
  - `first_name (second_name, LLVM version, DRM version, kernel_version)`
  - `first_name` is `radeon_info::marketing_name` or `radeon_info::name`
    - `marketing_name` is `amdgpu_get_marketing_name`
    - `name` is `enum radeon_family` with `CHIP_` prefix stripped
  - `second_name` is `radeon_info::lowercase_name`
    - this is `radeon_info:name` in lower csae
  - LLVM version is `MESA_LLVM_VERSION_STRING`
  - DRM version is `radeon_info::drm_major` and `radeon_info::drm_minor`
  - `kernel_version` is `utsname::release`
- radv
  - `VkPhysicalDeviceProperties::deviceName` is
    `radv_physical_device::marketing_name`
  - `%s (RADV %s%s)`
    - first string is `amdgpu_get_marketing_name` or `AMD Unknown`
    - second string is `radeon_info::name`
    - third string is `radv_get_compiler_string` and is empty unless llvm is
      used

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
