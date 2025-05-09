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

- DisplayPort 1.0
  - 2006
  - 10.8 Gb/s (HBR x 4 lanes)
  - DPCP (DisplayPort Content Protection)
- DisplayPort 1.1
  - 2007
  - alternative link layers such as fiber optic
  - HDCP (High-bandwidth Digital Content Protection) 1.3
  - DisplayPort Dual-Mode (DP++), allowing DVI and HDMI adapters
  - stereoscopic 3D
- DisplayPort 1.1a
  - 2008
- DisplayPort 1.2
  - 2010
  - 21.6 Gb/s (HBR2 x 4 lanes)
  - 4K at 60 Hz at 10 bpc
  - MST (Multi-Stream Transport)
  - more color spaces (sRGB, scRGB, DCI-P3)
  - compatible with Mini DisplayPort connector
- DisplayPort 1.2a
  - 2013
  - HDCP 2.2
  - optional Adaptive Sync
- DisplayPort 1.3
  - 2014
  - 32.4 Gb/s (HBR3 x 4 lanes)
  - 8K at 30 Hz
  - BT.2020
- DisplayPort 1.4
  - 2016
  - DSC (Display Stream Compression) 1.2
  - HDR10 with static and dynamic metadata
- DisplayPort 1.4a
  - 2018
  - DSC 1.2a
- DisplayPort 2.0
  - 2019
  - 80.0 Gb/s (UHBR 20 x 4 lanes)
- DisplayPort 2.1
  - 2022
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

## HDMI

- HDMI 1.0
  - 2002
  - 165 MHz / 3.96 Gbs
  - 1080p
- HDMI 1.1
  - 2004
  - DVD-Audio
- HDMI 1.2
  - 2005
  - PC-friendly by allowing RGB-only sources
- HDMI 1.2a
  - 2005
  - CEC, allowing control commands between devices
- HDMI 1.3
  - 2006
  - 340 MHz / 8.16 Gbs
  - 1440p
  - deep color (10, 12, and 16 bpc color depths)
  - cables
    - category 1: up to 74.25 MHz
    - category 2: up to 340 MHz
    - HDMI Type C Mini Connector
- HDMI 1.3a
  - 2006
- HDMI 1.4
  - 2009
  - 4K at 30 Hz
  - 3D
  - HEC (HDMI Ethernet Channel), allowing internet sharing
  - Micro HDMI Connector
- HDMI 1.4a
  - 2010
  - more 3D formats
- HDMI 1.4b
  - 2011
- HDMI 2.0
  - 2013
  - 600 MHz / 18.0 Gbs
  - 4K at 60 Hz
  - Rec. 2020 color space
- HDMI 2.0a
  - 2015
  - HDR with static metadata
- HDMI 2.0b
  - 2016
  - HDR10 and HLG
- HDMI 2.1
  - 2017
  - 48.0 Gbs
  - 4K at 120Hz
  - 8K at 60Hz
  - Ultra High Speed Cable
  - Dynamic HDR
  - DSC (Display Stream Compression) 1.2
  - HFR (High Frame Rate)
  - VRR (Variable Refresh Rate)
- HDMI 2.1a
  - 2022
  - SBTM (Source-Based Tone Mapping)

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
