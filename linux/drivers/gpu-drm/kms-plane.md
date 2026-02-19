DRM plane
=========

## `struct drm_plane_funcs`

- `update_plane` is legacy-only, and should be set to
  `drm_atomic_helper_update_plane` in atomic
- `disable_plane` is legacy-only, and should be set to
  `drm_atomic_helper_disable_plane` in atomic
- `destroy` is typically `drm_plane_cleanup`
  - `drmm_mode_config_init` calls `drm_mode_config_init_release`
    automatically, which calls this
- `reset` is typically `drm_atomic_helper_plane_reset`
  - during init or resume, `drm_mode_config_reset` calls this to reset
- `set_property` is legacy-only
- `atomic_duplicate_state` is typically
  `drm_atomic_helper_plane_duplicate_state`
  - during `drm_atomic_state` building, `drm_atomic_get_plane_state`
    calls this to duplicate `plane->state` as the new state for
    modification
- `atomic_destroy_state` is typically `drm_atomic_helper_plane_destroy_state`
  to destroy the duplicated state
- `atomic_set_property` is typically NULL
  - during commit or prop set, `drm_atomic_set_property` calls this for
    driver-specific props
- `atomic_get_property` is typically NULL
  - during prop get, `drm_atomic_get_property` calls this for driver-specific
    props
- `late_register` is typically NULL
  - `drm_plane_register_all` calls this for driver-specific userspace ifaces
- `early_unregister` is typically NULL
  - `drm_plane_unregister_all` calls this for driver-specific userspace ifaces
- `atomic_print_state` is typically NULL
  - during state dumping, `drm_atomic_plane_print_state` calls this to
    dump driver-specific states
- `format_mod_supported` is typically NULL
  - during atomic check (or init), `drm_plane_has_format` (or
    `create_in_format_blob`) calls this

## `struct drm_plane_helper_funcs`

- `prepare_fb` is typically NULL
  - during atomic commit, `drm_atomic_helper_prepare_planes` calls this or
    `drm_gem_plane_helper_prepare_fb` first
- `cleanup_fb` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_cleanup_planes` calls this
    last
- `begin_fb_access` is typically NULL
  - during atomic commit, `drm_atomic_helper_prepare_planes` calls this after
    `prepare_fb`
- `end_fb_access` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
- `atomic_check`
  - during atomic check, `drm_atomic_helper_check_planes` calls this
- `atomic_update`
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
- `atomic_enable` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
- `atomic_disable`
  - during atomic commit tail, `drm_atomic_helper_commit_planes` calls this
- `atomic_async_check`
  - during atomic check, `drm_atomic_helper_async_check` calls this
- `atomic_async_update`
  - during atomic submit, `drm_atomic_helper_async_commit` calls this
- `get_scanout_buffer` is typically NULL
  - during panic, `draw_panic_plane` calls this
- `panic_flush` is typically NULL
  - during panic, `draw_panic_plane` calls this
