DRM connector
=============

## `struct drm_connector_funcs`

- `dpms` is legacy-only
  - during atomic commit or prop set, crtc prop `ACTIVE` is updated and an
    atomic commit enables/disables the crtc instead
- `reset` is typically `drm_atomic_helper_connector_reset`
  - during init or resume, `drm_mode_config_reset` calls this to reset
- `detect`
  - during hpd irq, status poll, or `fill_modes`, `drm_helper_probe_detect`
    calls this to detect connector status
- `force` forces connector status (for debug)
  - `drm_helper_probe_single_connector_modes` calls this instead of `detect`
    when the status is forced
- `fill_modes` is typically `drm_helper_probe_single_connector_modes`
  - during userspace query, if master, `drm_mode_getconnector` calls this to
    update
    - `drm_connector::status`
    - `drm_connector::edid`
    - `drm_connector::modes`
- `set_property` is legacy-only
  - during prop set, `drm_mode_obj_set_property_ioctl` calls
    `atomic_set_property` instead when atomic
- `late_register` is typically NULL
  - `drm_connector_register` calls this for driver-specific userspace ifaces
- `early_unregister` is typically NULL
  - `drm_connector_unregister` calls this for driver-specific userspace
    ifaces
- `destroy` is typically NULL or `drm_connector_cleanup`
  - for static connectors initialized by `drmm_connector_init`,
    `drm_connector_cleanup_action` cleans up automatically
  - for dynamic connectors initialized by `drm_connector_dynamic_init`,
    `drm_connector_free` calls this to destroy when the refcount reaches 0
- `atomic_duplicate_state` is typically
  `drm_atomic_helper_connector_duplicate_state`
  - during `drm_atomic_state` building, `drm_atomic_get_connector_state`
    calls this to duplicate `connector->state` as the new state for
    modification
- `atomic_destroy_state` is typically
  `drm_atomic_helper_connector_destroy_state` to destroy the duplicated
  state
- `atomic_set_property` is typically NULL
  - during prop set, `drm_atomic_connector_set_property` calls this for
    driver-specific props
- `atomic_get_property` is typically NULL
  - during prop get, `drm_atomic_connector_get_property` calls this for
    driver-specific props
- `atomic_print_state` is typically NULL
  - during state dumping, `drm_atomic_connector_print_state` calls this to
    dump driver-specific states
- `oob_hotplug_event`
  - if usbc altmode detects hotplug oob, it calls
    `drm_connector_oob_hotplug_event` to notify drm
- `debugfs_init` is typically NULL
  - during `drm_connector_register`, `drm_debugfs_connector_add` calls this
    to add driver-specific files

## `struct drm_connector_helper_funcs`

- `get_modes`
  - during `drm_helper_probe_single_connector_modes`,
    `drm_helper_probe_get_modes` calls this to update
    `connector->probed_modes`
- `detect_ctx` replaces `detect`
- `mode_valid`
- `mode_valid_ctx` replaces `mode_valid`
  - after `get_modes`, `drm_connector_mode_valid` calls this to remove
    invalid modes
- `best_encoder`
- `atomic_best_encoder`
  - during atomic commit, `drm_atomic_helper_check` calls
    `drm_atomic_helper_check_modeset`, which calls `update_connector_routing`
    to call this
- `atomic_check`
  - after `atomic_best_encoder`, `drm_atomic_helper_check_modeset` calls this
    to check the atomic state
- `atomic_commit`
  - during atomic commit tail, `drm_atomic_helper_commit_writebacks` calls
    this for writeback
- `prepare_writeback_job`
  - during atomic commit, `drm_atomic_helper_prepare_planes` calls
    `drm_writeback_prepare_job` to call this
- `cleanup_writeback_job`
  - after hw writeback completion, `drm_writeback_signal_completion` calls
    `drm_writeback_cleanup_job` which calls this
- `enable_hpd`
  - on init or resume, `drm_kms_helper_poll_enable` calls
    `drm_kms_helper_enable_hpd` to call this
- `disable_hpd`
  - during suspend, `drm_kms_helper_poll_disable` calls
    `drm_kms_helper_disable_hpd` to call this
