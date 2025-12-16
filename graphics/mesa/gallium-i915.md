Gallium i915
============

## `i915g`

- `struct i915_screen`
  - buffer functions are initialized in `i915_init_screen_buffer_functions`
    - winsys is not used; It uses CPU memory (`malloc`)
    - `buffer_create` `malloc()`s the memory
    - `user_buffer_create` wraps a user buffer
    - there is no `surface_buffer_create`
  - texture functions are initialized in `i915_init_screen_texture_functions`
    - it uses `struct intel_winsys`, which in turn uses `libdrm_intel`
    - `texture_blanket` is not implemented
    - `get_tex_surface` and `get_tex_transfer` are cheap, as guessed
    - `transfer_map` maps the GEM object.
  - `struct drm_api` allows wrapping a GEM object in a `pipe_texture`
    - i915 pipe driver provides `i915_texture_blanket_intel`
    - Given the GEM object and the stride, it is wrapped in a `pipe_texture`.

