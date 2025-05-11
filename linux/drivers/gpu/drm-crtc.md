DRM crtc
========

## `struct drm_crtc_funcs`

- `reset` is typically `drm_atomic_helper_crtc_reset`
- `cursor_set` is deprecated
- `cursor_set2` is deprecated
- `cursor_move` is deprecated
- `gamma_set` is deprecated
- `destroy` is typically `drm_crtc_cleanup`
- `set_config` is legacy-only and should be set to
  `drm_atomic_helper_set_config` on atomic
- `page_flip` is legacy-only
- `page_flip_target` is legacy-only
- `set_property` is legacy-only
- `atomic_duplicate_state`
- `atomic_destroy_state`
- `atomic_set_property` is typically NULL
- `atomic_get_property` is typically NULL
- `late_register` is typically NULL
- `early_unregister` is typically NULL
- `set_crc_source` is typically NULL
- `verify_crc_source` is typically NULL
- `get_crc_sources` is typically NULL
- `atomic_print_state` is typically NULL
- `get_vblank_counter` is typically NULL
- `enable_vblank`
- `disable_vblank`
- `get_vblank_timestamp`

## `struct drm_crtc_helper_funcs`

- `dpms` is legacy-only, and is deprecated by `atomic_enable`/`atomic_disable`
- `prepare` is legacy-only, and is deprecated by `atomic_disable`
- `commit` is legacy only, and is depreacated by `atomic_enable`
- `mode_valid`
- `mode_fixup` is deprecated by `atomic_check`
- `mode_set` is deprecated
- `mode_set_nofb` is typicall NULL
- `mode_set_base` is deprecated
- `mode_set_base_atomic` is typically NULL
- `disable`
- `atomic_check`
- `atomic_begin`
- `atomic_flush`
- `atomic_enable`
- `atomic_disable`
- `get_scanout_position`
