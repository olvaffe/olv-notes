DRM Panel
=========

## Panel Driver

- connector types
  - <https://en.wikipedia.org/wiki/List_of_video_connectors>
  - `DRM_MODE_CONNECTOR_DisplayPort`
  - `DRM_MODE_CONNECTOR_HDMIA`
  - `DRM_MODE_CONNECTOR_HDMIB` (dual-link)
  - `DRM_MODE_CONNECTOR_eDP`
  - `DRM_MODE_CONNECTOR_VIRTUAL`
  - `DRM_MODE_CONNECTOR_DSI` (mobile)
  - `DRM_MODE_CONNECTOR_DPI` (mobile, low-res)
  - `DRM_MODE_CONNECTOR_WRITEBACK`
  - `DRM_MODE_CONNECTOR_SPI` (mobile, very low-res epaper)
  - `DRM_MODE_CONNECTOR_USB` (apple touchbar)
  - legacy pc/mobile connectors
    - `DRM_MODE_CONNECTOR_LVDS`
  - legacy pc connectors
    - `DRM_MODE_CONNECTOR_VGA`
    - `DRM_MODE_CONNECTOR_DVI*`
  - legacy tv connectors
    - `DRM_MODE_CONNECTOR_Composite`
    - `DRM_MODE_CONNECTOR_SVIDEO`
    - `DRM_MODE_CONNECTOR_Component`
    - `DRM_MODE_CONNECTOR_9PinDIN`
    - `DRM_MODE_CONNECTOR_TV` (nv only)
- basic flow
  - `drm_panel_init` inits a `drm_panel` with `drm_panel_funcs` and connector
    type
  - `drm_panel_add` adds the panel to global `panel_list`
  - `devm_drm_panel_alloc` combines alloc, init, and add
- `drm_panel_funcs` callbacks
  - `prepare` / `unprepare` inits the panel hw
  - `enable` / `disable` enables backlight, etc.
  - `get_*` queries modes, timing, etc.

## Display Driver

- `of_drm_find_panel` finds the panel from `panel_list`
  - the dt node usually has a child node or a port refering to the panel
- more commonly, `devm_drm_of_get_bridge` or `drmm_of_get_bridge` returns a
  bridge with potential wrapping
  - `drm_of_find_panel_or_bridge` calls `of_drm_find_panel`, or falls back
    to `of_drm_find_bridge`
  - if panel, `devm_drm_panel_bridge_add` or `drmm_panel_bridge_add` wraps
    the panel inside a bridge
  - this allows the display driver to deal with bridge/panel uniformly

## MIDI DSI

- the display driver calls `mipi_dsi_host_register` to register a host
  - `mipi_dsi_host_register` scans all dt child nodes and calls
    `of_mipi_dsi_device_add`
- the panel driver has `module_mipi_dsi_driver` or calls
  `mipi_dsi_driver_register`
  - on probe, driver calls `mipi_dsi_attach` to attach the device to the host
- the panel driver sends messages to the panel
  - the message could be standardized DCS (display command set) or
    non-standard MCS (manufacturer command set)
  - for dcs, `mipi_dsi_dcs_*` calls `mipi_dsi_dcs_write`
    - `mipi_dsi_dcs_write_buffer`
    - `mipi_dsi_device_transfer`
      - the display driver calls `mipi_dsi_create_packet` to wrap the message
        in a packet and sends the packet over to the panel

## DP

- DP supports aux channel to communicate with the panel
- the display driver
  - `drm_dp_aux_init` inits an aux channel
  - `devm_of_dp_aux_populate_bus` adds the device on the aux channel
    - it finds `aux-bus` child node and uses the first child node of `aux-bus`
  - `drm_dp_aux_register` registers the aux channel
    - it also calls `i2c_add_adapter` to add a i2c adapter for ddc, used with
      `drm_edid_read_ddc` to read edid
  - the display driver provides a transfer function for communication over the
    channel
    - it is used to access DPCD (displayport configuration data)
    - the i2c adapter simply wraps `i2c_msg` in `drm_dp_aux_msg`
- the panel driver
  - `dp_aux_dp_driver_register` registers the driver to `dp-aux` bus
  - `drm_panel_dp_aux_backlight` is the std way to register a backlight device
    - `drm_dp_dpcd_read_data(DP_EDP_DPCD_REV)` returns caps
    - if panel has intrinsic bl support,
      - `drm_edp_backlight_init` reads more caps
      - `devm_backlight_device_register` registers the bl dev with
        `dp_aux_bl_ops`
