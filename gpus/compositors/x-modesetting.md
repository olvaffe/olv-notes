X Modesetting Driver
====================

## Atomic

- atomic modesetting is disabled by kernel
  - see `DRM_CLIENT_CAP_ATOMIC`
- because atomic is disabled by kernel, there is no KMS modifier support

## DRI3

- `DRI3GetSupportedModifiers` asks for screen modifiers and drawable modifiers
- screen modifiers are handled and cached by `cache_formats_and_modifiers`
  - glamor returns no format nor modifier by default
    - need to enable manually
  - `glamor_get_formats` calls `eglQueryDmaBufFormatsEXT` to get screen
    formats
  - `glamor_get_modifiers` calls `eglQueryDmaBufModifiersEXT` to get screen
    modifiers
    - on intel gen9 with iris, this returns
      - `DRM_FORMAT_MOD_LINEAR`
      - `I915_FORMAT_MOD_X_TILED`
      - `I915_FORMAT_MOD_Y_TILED`
      - `I915_FORMAT_MOD_Y_TILED_CCS`
- drawable modifiers are an intersection of what EGL and KMS can support
  - `glamor_get_drawable_modifiers` calls `get_drawable_modifiers` in
    modesetting driver
  - the modesetting driver returns the modifiers set up by
    `populate_format_modifiers`, which parses KMS `IN_FORMATS`
    - on intel gen9, this returns
      - `DRM_FORMAT_MOD_LINEAR`
      - `I915_FORMAT_MOD_X_TILED`
      - `I915_FORMAT_MOD_Y_TILED`
      - `I915_FORMAT_MOD_Y_TILED_CCS`
      - `I915_FORMAT_MOD_Yf_TILED`
      - `I915_FORMAT_MOD_Yf_TILED_CCS`
  - `dri3_get_supported_modifiers` then calculates the intersection

## Modifiers

- modifiers require glamor and modesetting support
  - to enable support in glamor, manually set `dmabuf_capable` and make sure
    EGL has `EGL_EXT_image_dma_buf_import` and
    `EGL_EXT_image_dma_buf_import_modifiers`
  - to enable support in modesetting, manually set `DRM_CLIENT_CAP_ATOMIC` to 2
    - this enables atomic which is broken in many cases
- only non-composited fullscreen window advertises modifiers
  - this is because `get_drawable_modifiers` in modesetting checks
    `present_can_window_flip`, which returns true when the window is
    fullscreen (and other conditions)
