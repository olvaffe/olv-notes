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

## eDP panel driver

- DP supports aux channel to communicate with the panel
- `dp_aux_dp_driver_register` registers the driver to `dp-aux` bus
- `drm_panel_dp_aux_backlight` is the std way to register a backlight device
