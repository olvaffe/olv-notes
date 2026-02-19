DRM Panel
=========

## Common Resolutions

- tv standards
  - HD, 1280x720
  - FHD, 1920x1080
  - QHD, 2560x1440
  - 4K UHD, 3840x2160
  - 8K UHD, 7680x4320
- pc standards
  - VGA, 640x480
  - SVGA, 800x600
  - XGA, 1024x768
  - SXGA, 1280x1024
  - UXGA, 1600x1200

## EDID

- 128-byte structure, optionally followed by other 128-byte extensions
- EDID structure, version 1.4
  - byte 0..19: manufacturer id, product code, serial, year, week, EDID ver
  - byte 20..24: basic dispaly params (bit depth, hdmi/dp, monitor size, etc)
  - byte 25..34: xy coordinates of rgbw
  - byte 35..37: supported (outdated) modes
  - byte 38..53: up to eight 2-byte display mode descriptions
  - byte 54..125: four 18-byte timing/display descriptors
  - byte 126: number of extensions
  - byte 127: checksum

## DisplayPort

- 2006, DisplayPort 1.0
  - 10.8 Gb/s (HBR x 4 lanes)
  - DPCP (DisplayPort Content Protection)
- 2007, DisplayPort 1.1
  - alternative link layers such as fiber optic
  - HDCP (High-bandwidth Digital Content Protection) 1.3
  - DisplayPort Dual-Mode (DP++), allowing DVI and HDMI adapters
  - stereoscopic 3D
- 2008, DisplayPort 1.1a
- 2010, DisplayPort 1.2
  - 21.6 Gb/s (HBR2 x 4 lanes)
  - 4K at 60 Hz at 10 bpc
  - MST (Multi-Stream Transport)
  - more color spaces (sRGB, scRGB, DCI-P3)
  - compatible with Mini DisplayPort connector
- 2013, DisplayPort 1.2a
  - optional Adaptive Sync and PSR (panel self-refresh)
- 2014, DisplayPort 1.3
  - 32.4 Gb/s (HBR3 x 4 lanes, 8K@30Hz)
  - mandatory Dual-Mode for DVI and HDMI (2.0) adapters
    - HDMI CEC is tunneled via AUX
  - HDCP 2.2
- 2016, DisplayPort 1.4
  - DSC (Display Stream Compression) 1.2
  - HDR10 with static and dynamic metadata
  - BT.2020
- 2018, DisplayPort 1.4a
  - DSC 1.2a
- 2019, DisplayPort 2.0
  - 80.0 Gb/s (UHBR20 x 4 lanes, 8K@60Hz)
- 2022, DisplayPort 2.1
  - DP40 cable: up to UHBR10x4
  - DP80 cable: up to UHBR20x4
- 2024, DisplayPort 2.1a
  - DP54 cable: up to UHBR13.5x4, replacing DP40
- 2025, DisplayPort 2.1b
- eDP
  - eDP 1.0, 2008
  - eDP 1.1/1.1a, 2009
  - eDP 1.2, 2010
  - eDP 1.3, 2011
    - PSR (Panel Self-Refresh)
  - eDP 1.4, 2013
    - based on DP 1.2
  - eDP 1.4a, 2015
    - based on DP 1.3
  - eDP 1.4b, 2015
  - eDP 1.5, 2021
- DisplayID, an EDID replacement
- Dual Mode
  - when a dp connector supports dual-mode, it internally connects to both a
    dp phy and an hdmi phy
  - the display controller will output dp signal or hdmi signal depending on
    whether the sink is dp or hdmi
  - as such, dp-to-hdmi cable is a passive adapter and is simple
- type-c dp alt mode
  - type-c has 4 differential pairs
  - when a pair (x1) or two pairs (x2) are configured as dp lanes, it can
    provide both DP and SS usb at the same time
  - when all pairs (x4) are configured as dp lanes, it can provide both DP and
    non-SS usb at the same time
  - AUX is supported
  - PD is supported simultaneously
  - no dual mode
    - but there is type-c hdmi alt mode to provide hdmi (instead of dp)

## HDMI

- 2002, HDMI 1.0
  - 165 MHz / 3.96 Gbs (1080p@60Hz)
- 2004, HDMI 1.1
  - DVD-Audio
- 2005, HDMI 1.2
  - PC-friendly by allowing RGB-only sources
- 2005, HDMI 1.2a
  - CEC, allowing control commands between devices
- 2006, HDMI 1.3
  - 340 MHz / 8.16 Gbs (1440p@75Hz, 1080p@144Hz)
  - deep color (10, 12, and 16 bpc color depths)
  - mini hdmi (type c; unrelated to usb type-c)
  - standard cable: category 1, up to 74.25 MHz
  - high speed cable: category 2, up to 340 MHz
- 2006, HDMI 1.3a
- 2009, HDMI 1.4
  - 4K@30Hz
  - 3D
  - HEC (HDMI Ethernet Channel), allowing internet sharing
  - micro hdmi (type d)
- 2010, HDMI 1.4a
  - more 3D formats
- 2011, HDMI 1.4b
- 2013, HDMI 2.0
  - 600 MHz / 14.4 Gbs (4K@60Hz)
  - Rec. 2020 color space
  - premium high speed cable: up to 600 MHz
- 2015, HDMI 2.0a
  - HDR with static metadata
- 2016, HDMI 2.0b
  - HDR10 and HLG
- 2017, HDMI 2.1
  - not allowed in open source
  - 600 MHz / 42.0 Gbs (8K@60Hz, 4K@120Hz)
  - ultra high speed cable: up to 42.0 Gbs
  - Dynamic HDR
  - DSC (Display Stream Compression) 1.2
  - HFR (High Frame Rate)
  - VRR (Variable Refresh Rate)
- 2022, HDMI 2.1a
  - SBTM (Source-Based Tone Mapping)
- 2023, HDMI 2.1b
- 2025, HDMI 2.2
  - 600 MHz / 84.0 Gbs (8K@60Hz, 4K@120Hz)
  - Latency Indication Protocol (LIP)

## `struct drm_panel_funcs`

- `prepare`
  - `panel_bridge_atomic_pre_enable` calls `drm_panel_prepare` to call this
- `enable`
  - `panel_bridge_atomic_enable` calls `drm_panel_enable` to call this
- `disable`
  - `panel_bridge_atomic_disable` calls `drm_panel_disable` to call this
- `unprepare`
  - `panel_bridge_atomic_post_disable` calls `drm_panel_unprepare` to call
    this
- `get_modes`
  - `panel_bridge_get_modes` calls `drm_panel_get_modes` to call this
  - we assume `DRM_BRIDGE_ATTACH_NO_CONNECTOR` because other paths are
    deprecated
- `get_orientation` typically calls `of_drm_get_panel_orientation`
  - `drm_bridge_connector_init` calls `drm_panel_bridge_set_orientation` which
    calls `drm_connector_set_orientation_from_panel` to call this
- `get_timings` is unused
- `debugfs_init` is typically NULL
  - `panel_bridge_debugfs_init` calls this

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
