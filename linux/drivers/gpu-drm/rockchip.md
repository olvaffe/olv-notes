Linux DRM Rockchip
==================

## RK3588

- the relevant dt nodes are
  - `vop`, the 2d overlay engine
    - it has 4 ports, `vp0` to `vp3`
  - `hdmi0`, the hdmi controller (downstream only)
    - it has 2 ports, `hdmi0_in` and `hdmi0_out`
  - `hdptxphy_hdmi0`, the hdmi phy
  - `hdmi0-con`, the hdmi connector (downstream only)
    - it has one port, `hdmi0_con_in`
- the dt graph on opi5
  - `vp0` connects to `hdmi0_in` with reg `ROCKCHIP_VOP2_EP_HDMI0`
  - `hdmi0_out` connects to `hdmi0_con_in`
- `CONFIG_DRM_DISPLAY_CONNECTOR`
  - when the dt has this node for the hdmi connector
    - `compatible = "hdmi-connector";`
    - `type = "a";`
  - `display_connector_probe` probes the node independent of rockchip
    - the connector type is `DRM_MODE_CONNECTOR_HDMIA`
    - `drm_bridge_add` adds the connector as a `drm_bridge` to `bridge_list`

## Initialization

- the dt has a node for `rockchip,display-subsystem`
  - `ports` refers to `vop` ports
- `rockchip_drm_init` registers the driver and all subdrivers
  - `rockchip_drm_platform_driver`
  - `vop2_platform_driver`
  - `dw_hdmi_qp_rockchip_pltfm_driver` (downstream only)
- `rockchip_drm_platform_probe`
  - `rockchip_drm_platform_of_probe` checks for `ports`
  - `rockchip_drm_match_add` builds `component_match` for all subdriver
    platform devices
  - `component_master_add_with_match` makes sure all subdrivers have probed
    their platform devices before calling `rockchip_drm_bind`
- `rockchip_drm_bind` inits the `drm_device`
  - `component_bind_all` binds all subdrivers
  - `rockchip_drm_init_iommu` inits iommu
  - `drm_dev_register` registers the drm dev
  - `drm_fbdev_dma_setup` sets up fbdev emu

## `CONFIG_ROCKCHIP_VOP2`

- when the dt has this node for the vop
  - `compatible = "rockchip,rk3588-vop";`
  - `iommus = <&vop_mmu>;`
  - there are 4 ports (crtcs), `vp0` to `vp3`
- `vop2_bind` binds the node when the main driver calls `component_bind_all`
  - `rk3588_vop`
    - there are 4 `vop2_video_port_data`
    - there are 8 `vop2_win_data`
      - 4 clusters
      - 4 esmarts
  - `vop2_win_init` inits `vop2->win` array
  - `vop2_create_crtcs` inits `vop2->vps` array
    - assuming only `vp0` is connected, then
    - `vp0` is the only crtc
    - `vop2_plane_init` inits planes
      - first cluster is tied to `vp0`, as its primary plane and with `vp0` as
        the only possible crtc
      - the remaining clusters and all esmarts are overlay planes, and they
        can be used with all crtcs
    - `drm_crtc_init_with_planes` inits `vp0` as the crtc
  - `rockchip_drm_dma_init_device` makes the vop device the iommu dev

## `CONFIG_ROCKCHIP_DW_HDMI_QP` (downstream only)

- when the dt has this node for the hdmi controller
  - `compatible = "rockchip,rk3588-dw-hdmi-qp";`
  - there are two ports
    - `hdmi0_in` connects to vop
    - `hdmi0_out` connects to hdmi-connector
- `dw_hdmi_qp_rockchip_bind` binds the node when the main driver calls
  `component_bind_all`
  - `rk3588_hdmi_phy_ops` is the phy ops
  - a `drm_encoder` is created and initialized
    - `drm_of_find_possible_crtcs` returns the possible crtcs
    - `dw_hdmi_qp_rockchip_encoder_helper_funcs` is the helper funcs
    - `drm_simple_encoder_init` inits with `DRM_MODE_ENCODER_TMDS`
      - this adds the encoder to the `drm_device`
  - `dw_hdmi_qp_bind` creates a `dw_hdmi_qp`
    - it is a subclass of a `drm_bridge`
    - bridge type `DRM_MODE_CONNECTOR_HDMIA`
    - supported ops are
      - `DRM_BRIDGE_OP_DETECT`
      - `DRM_BRIDGE_OP_EDID`
      - `DRM_BRIDGE_OP_HDMI`
      - `DRM_BRIDGE_OP_HPD`
    - `devm_drm_bridge_add` adds the brdige to `bridge_list`
    - `drm_bridge_attach` attaches the bridge to the encoder
  - `drm_bridge_connector_init` creates a `drm_connector`
    - `dw_hdmi_qp` is the only bridge
    - connector ops are passed through to the bridge
    - `drmm_connector_hdmi_init` adds the connector to the `drm_device`
  - `drm_connector_attach_encoder` attaches the connector to the encoder
    - this sets `connector->possible_encoders`
