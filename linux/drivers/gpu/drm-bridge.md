Linux DRM Bridge
================

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
