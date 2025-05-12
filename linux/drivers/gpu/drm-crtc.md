DRM crtc
========

## `struct drm_crtc_funcs`

- `reset` is typically `drm_atomic_helper_crtc_reset`
  - during init or resume, `drm_mode_config_reset` calls this to reset
- `cursor_set` is deprecated
- `cursor_set2` is deprecated
- `cursor_move` is deprecated
- `gamma_set` is deprecated
- `destroy` is typically `drm_crtc_cleanup`
  - `drmm_mode_config_init` calls `drm_mode_config_init_release`
    automatically, which calls this
- `set_config` is legacy-only and should be set to
  `drm_atomic_helper_set_config` on atomic
- `page_flip` is legacy-only
- `page_flip_target` is legacy-only
- `set_property` is legacy-only
- `atomic_duplicate_state` is typically
  `drm_atomic_helper_crtc_duplicate_state`
  - during `drm_atomic_state` building, `drm_atomic_get_crtc_state`
    calls this to duplicate `crtc->state` as the new state for
    modification
- `atomic_destroy_state` is typically `drm_atomic_helper_crtc_destroy_state`
  to destroy the duplicated state
- `atomic_set_property` is typically NULL
  - during commit or prop set, `drm_atomic_set_property` calls this for
    driver-specific props
- `atomic_get_property` is typically NULL
  - during prop get, `drm_atomic_get_property` calls this for driver-specific
    props
- `late_register` is typically NULL
  - `drm_crtc_register_all` calls this for driver-specific userspace ifaces
- `early_unregister` is typically NULL
  - `drm_crtc_unregister_all` calls this for driver-specific userspace ifaces
- `set_crc_source` is typically NULL
- `verify_crc_source` is typically NULL
- `get_crc_sources` is typically NULL
- `atomic_print_state` is typically NULL
  - during state dumping, `drm_atomic_crtc_print_state` calls this to
    dump driver-specific states
- `get_vblank_counter` is typically NULL
  - this is to ensure `drm_get_last_vbltimestamp` is correct
- `enable_vblank`
  - when vblank irq is needed, `drm_vblank_enable` calls this
- `disable_vblank`
  - when vblank irq is not needed, `drm_vblank_disable` calls this
- `get_vblank_timestamp` is typically NULL or
  `drm_crtc_vblank_helper_get_vblank_timestamp`
  - `drm_crtc_get_last_vbltimestamp` calls this to get high-precision
    timestamp of the last vblank

## `struct drm_crtc_helper_funcs`

- `dpms` is legacy-only, and is deprecated by `atomic_enable`/`atomic_disable`
- `prepare` is legacy-only, and is deprecated by `atomic_disable`
- `commit` is legacy only, and is depreacated by `atomic_enable`
- `mode_valid`
  - during atomic check or connector `get_modes`, `drm_crtc_mode_valid` calls
    this to check if a mode is valid
- `mode_fixup` is deprecated by `atomic_check`
- `mode_set` is deprecated
- `mode_set_nofb` is typicall NULL
  - `drm_atomic_helper_commit_modeset_disables` calls this
- `mode_set_base` is deprecated
- `mode_set_base_atomic` is typically NULL
  - during kgdb, `drm_fb_helper_debug_enter` and `drm_fb_helper_debug_leave`
    calls this
- `disable`
- `atomic_check`
  - during atomic check, `drm_atomic_helper_check_planes` calls this
- `atomic_begin`
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
    before plane enable/disable
- `atomic_flush`
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
    after plane enable/disable
- `atomic_enable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_enables`
    calls this (over `commit`)
- `atomic_disable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls `disable_outputs` which calls this (over `prepare`, `disable`, or
    `dpms`)
- `get_scanout_position`
  - `drm_crtc_vblank_helper_get_vblank_timestamp` calls this to derive
    high-precision timestamp of the last vblank
