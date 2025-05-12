Linux DRM Bridge
================

## `struct drm_bridge_funcs`

- `attach`
  - during encoder building, `drm_bridge_attach` calls this
- `detach`
  - during encoder cleanup, `drm_encoder_cleanup` calls `drm_bridge_detach` to
    call this
- `mode_valid`
  - during atomic check or connector `get_modes`,
    `drm_bridge_chain_mode_valid` calls this
- `mode_fixup` is deprecated by `atomic_check`
- `disable` is deprecated by `atomic_disable`
- `post_disable` is deprecated by `atomic_post_disable`
- `mode_set` is deprecated by `atomic_enable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls `drm_bridge_chain_mode_set` to call this
- `pre_enable` is deprecated by `atomic_pre_enable`
- `enable` is deprecated by `atomic_enable`
- `atomic_pre_enable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_enables`
    calls `drm_atomic_bridge_chain_pre_enable` which calls
    `drm_atomic_bridge_call_pre_enable` to call this
- `atomic_enable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_enables`
    calls `drm_atomic_bridge_chain_enable` to call this
- `atomic_disable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls `drm_atomic_bridge_chain_disable` to call this
- `atomic_post_disable`
  - during atomic commit tail, `drm_atomic_helper_commit_modeset_disables`
    calls `drm_atomic_bridge_chain_post_disable` to call
    `drm_atomic_bridge_call_post_disable` which calls this
- `atomic_duplicate_state` is typically
  `drm_atomic_helper_bridge_duplicate_state`
  - during encoder building, `drm_bridge_attach` calls
    `drm_atomic_private_obj_init` with
    `drm_bridge_atomic_duplicate_priv_state` to call this
- `atomic_destroy_state`
  - during encoder building, `drm_bridge_attach` calls
    `drm_atomic_private_obj_init` with
    `drm_bridge_atomic_destroy_priv_state` to call this
- `atomic_get_output_bus_fmts`
  - during atomic check, `drm_atomic_helper_check_modeset` calls
    `drm_atomic_bridge_chain_check` which calls
    `drm_atomic_bridge_chain_select_bus_fmts` to call this
- `atomic_get_input_bus_fmts`
  - during atomic check, `drm_atomic_helper_check_modeset` calls
    `drm_atomic_bridge_chain_check` which calls
    `drm_atomic_bridge_chain_select_bus_fmts` to call this
- `atomic_check`
  - during atomic check, `drm_atomic_helper_check_modeset` calls
    `drm_atomic_bridge_chain_check` which calls `drm_atomic_bridge_check` to
    call this
- `atomic_reset` is typically `drm_atomic_helper_bridge_reset`
  - during encoder building, `drm_bridge_attach` calls this
- `detect`
  - `drm_bridge_connector_detect` calls this
- `get_modes`
  - `drm_bridge_connector_get_modes` calls this
- `edid_read`
  - `drm_bridge_connector_get_modes` calls `drm_bridge_edid_read` to call this
- `hpd_notify` is typically NULL
  - during hpd irq or detect, `drm_bridge_connector_hpd_notify` calls this
- `hpd_enable`
  - `drm_bridge_connector_enable_hpd` calls `drm_bridge_hpd_enable` to call
    this
- `hpd_disable`
  - `drm_bridge_connector_disable_hpd` calls `drm_bridge_hpd_disable` to call
    this
- `hdmi_clear_infoframe`
  - `drm_bridge_connector_clear_infoframe` calls this
- `hdmi_write_infoframe`
  - `drm_bridge_connector_write_infoframe` calls this
- `hdmi_audio_startup`
  - `drm_bridge_connector_audio_startup` calls this
- `hdmi_audio_prepare`
  - `drm_bridge_connector_audio_prepare` calls this
- `hdmi_audio_shutdown`
  - `drm_bridge_connector_audio_shutdown` calls this
- `hdmi_audio_mute_stream`
  - `drm_bridge_connector_audio_mute_stream` calls this
- `dp_audio_startup`
  - `drm_bridge_connector_audio_startup` calls this
- `dp_audio_prepare`
  - `drm_bridge_connector_audio_prepare` calls this
- `dp_audio_shutdown`
  - `drm_bridge_connector_audio_shutdown` calls this
- `dp_audio_mute_stream`
  - `drm_bridge_connector_audio_mute_stream` calls this
- `debugfs_init`
  - `drm_bridge_connector_debugfs_init` calls this

## Bridge Driver

- all bridge drivers should
  - fill in `drm_bridge_funcs`
  - call `devm_drm_bridge_add` to add the bridge to global `bridge_list`
- most bridge drivers expect a downstream bridge
  - the downstream bridge can be
    - a real `drm_bridge`
    - a `drm_panel` wrapped in a `drm_bridge`
    - a physical connector wrapped in a `drm_bridge`
  - the bridge driver should call one of `devm_drm_of_get_bridge`,
    `of_drm_find_bridge`, or `drm_of_find_panel_or_bridge` to find the
    downstream
- some bridge drivers do not expect a downstream bridge
  - `CONFIG_DRM_PANEL_BRIDGE` is a `drm_panel` wrapped in a `drm_bridge`, and
    does not expect any downstream
  - `CONFIG_DRM_DISPLAY_CONNECTOR` is a physical connector wrapped in a
    `drm_bridge`, and does not expect any downstream
  - some real bridge drivers expect a physical connector and do not make use
    of `CONFIG_DRM_DISPLAY_CONNECTOR`
  - legacy display driver expects such bridge drivers to create a
    `drm_connector`
    - new display driver specifies `DRM_BRIDGE_ATTACH_NO_CONNECTOR` to avoid
      the step
- DSI-to-eDP bridge driver
  - the bridge is both a DSI sink and a DP source
  - the bridge driver is somewhat like a DSI panel driver
    - the dt does not describe the bridge as a child of the DSI output
      - `mipi_dsi_host_register` does not call `of_mipi_dsi_device_add`
        automatically
      - instead, the bridge driver calls `devm_mipi_dsi_device_register_full`
        explicitly
    - there is no DSI panel driver, but the bridge driver calls
      `mipi_dsi_attach` directly
  - the bridge driver is somewhat like a eDP output driver
    - `drm_dp_aux_init` inits a DP aux channel, `drm_dp_aux`
    - `devm_of_dp_aux_populate_bus` adds the eDP downstream device
      - this expects a `aux-bus` subnode and uses its first child as the device
        - there is no device discovery
      - the dev is added to `dp-aux` bus for driver matching
      - `done_probing` is called after a driver probes successfully

## Display Driver

- at a high-level,
  - a `drm_crtc` reads pixel data from `drm_plane`s, processes the pixel data,
    and outputs processed pixel data
  - a `drm_encoder` reads pixel data from a `drm_crtc` and outputs encoded
    pixel signal
  - a `drm_connector` passes pixel signal from `drm_encoder` to a display
  - in soc design, a chain of bridges reads pixel data from a `drm_crtc` and
    outputs pixel signal to a display
    - both `drm_encoder` and `drm_connector` represent the entire chain of
      bridges
    - `drm_encoder` is more like the head of the chain, with ops to configure
      the chain itself
    - `drm_connector` is more like the tail of the chain, with ops to control
      the display
      - e.g., detect connection, query edid, etc.
- for each output pipeline,
  - `drm_encoder_init` inits a `drm_encoder`
  - `devm_drm_of_get_bridge` or `drmm_of_get_bridge` returns a bridge, or a
    panel wrapped in a bridge
  - `drm_bridge_attach` adds the head of the bridge chain to the encoder
    - a `drm_encoder` is essentially a chain of bridges
  - `drm_bridge_connector_init` creates a `drm_connector` from the chain of
    bridges
  - `drm_connector_attach_encoder` adds the encoder to the connector
- an eDP controller is similar to an any-to-eDP bridge
  - `drm_dp_aux_init` inits a DP aux channel
  - `devm_of_dp_aux_populate_bus` adds the downstream device
  - after a dp-aux driver binds to the device, `done_probing` is called
    - `devm_drm_of_get_bridge` finds the device
    - `devm_drm_bridge_add` adds self as a bridge
  - when the pixel source driver calls `drm_bridge_attach`, `attach` is called
    - `drm_dp_aux_register` registers the DP aux channel
    - `drm_bridge_attach` attaches the downstream device
  - when the pixel souce driver calls `drm_bridge_connector_init`, it inits a
    connector for the entire bridge chain
  - later, when `drm_atomic_helper_commit_tail` commits,
    `drm_atomic_bridge_chain_enable` enables all bridges
