Android hwui
============

## Properties

- `HWUIProperties.sysprop`
  - <https://source.android.com/docs/core/architecture/configuration/sysprops-apis>
  - `ro.hwui.use_vulkan`
  - `ro.hwui.render_ahead`
- `Properties.h` defines debug properties
  - `PROPERTY_DEBUG` is `debug.hwui.level`
  - `PROPERTY_USE_BUFFER_AGE` is `debug.hwui.use_buffer_age`
  - `PROPERTY_ENABLE_PARTIAL_UPDATES` is `debug.hwui.use_partial_updates`
  - `PROPERTY_RENDERER` is `debug.hwui.renderer`
- `Properties::peekRenderPipelineType` returns `SkiaGL` or `SkiaVulkan`
  - it is determined by `debug.hwui.renderer`
    - if `ro.hwui.use_vulkan` is true, it defaults to `skiavk`
    - otherwise, it defaults to `skiagl`
