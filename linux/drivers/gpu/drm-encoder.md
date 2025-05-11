DRM encoder
===========

## `struct drm_encoder_funcs`

- `reset` is typically NULL
  - during init or resume, `drm_mode_config_reset` calls this to reset
- `destroy` is typically `drm_encoder_cleanup`
  - `drmm_mode_config_init` calls `drm_mode_config_init_release`
    automatically, which calls this
- `late_register` is typically NULL
  - `drm_encoder_register_all` calls this for driver-specific userspace ifaces
- `early_unregister` is typically NULL
  - `drm_encoder_unregister_all` calls this for driver-specific userspace
    ifaces
- `debugfs_init` is typically NULL
  - during `drm_encoder_register_all`, `drm_debugfs_encoder_add` calls this to
    add driver-specific files

## `struct drm_encoder_helper_funcs`

- `dpms` is legacy-only, and is deprecated by `enable`/`disable` in atomic
- `mode_valid` is typically NULL
  - during atomic check or connector `get_modes`, `drm_encoder_mode_valid`
    calls this to check if a mode is valid
- `mode_fixup` is deprecated by `atomic_check`
- `prepare` is legacy-only, and is deprecated by `disable` in atomic
- `commit` is legacy only, and is depreacated by `enable` in atomic
- `mode_set`
- `atomic_mode_set` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls this
- `detect` is deprecated
- `atomic_disable` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls this (over `prepare`, `disable`, or `dpms`)
- `atomic_enable` is typically NULL
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_enables`
    calls this (over `enable` or `commit`)
- `disable`
- `enable`
- `atomic_check` is typically NULL
  - during atomic check, `mode_fixup` calls this instead of `mode_fixup` when
    defined
