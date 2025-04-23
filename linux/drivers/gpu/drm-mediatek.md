Linux DRM Mediatek
==================

## Initialization

- `CONFIG_MTK_MMSYS` adds the platform devices
  - MMSYS stands for multimedia subsystem
  - on mt8195, there are two display pipelines
    - `VDOSYS0`, `mediatek,mt8195-vdosys0`
      - it supports PQ (COLOR, CCORR, AAL, GAMMA, and DITHER)
    - `VDOSYS1`, `mediatek,mt8195-vdosys1`
      - it supports HDR (ETHDR)
    - two platform devices are added
- `mtk_drm_platform_driver` binds to the two platform devices on mt8195
  - the compat strings of the OF nodes are matched again to get
    `mtk_mmsys_driver_data`
    - `mt8195_vdosys0_driver_data` has `main_path`
      - `DDP_COMPONENT_OVL0` blends planes
      - `DDP_COMPONENT_RDMA0` stands for Read Direct Memory Access
      - `DDP_COMPONENT_COLOR0`
        - `mtk_color_config` configs width/height
      - `DDP_COMPONENT_CCORR`
        - `mtk_ccorr_ctm_set` applies a 3x3 color matrix
      - `DDP_COMPONENT_AAL0` stands for Adaptive Ambient Light
      - `DDP_COMPONENT_GAMMA`
        - `mtk_gamma_set` applies a gamma lut
      - `DDP_COMPONENT_DITHER0`
        - `mtk_dither_config` applies dither
      - `DDP_COMPONENT_DSC0` is VESA Display Stream Compression
      - `DDP_COMPONENT_MERGE0` merges two inputs into side-by-side output?
      - `DDP_COMPONENT_DP_INTF0` outputs RGB/YUV signal
    - `mt8195_vdosys1_driver_data` has `ext_path`
      - `DDP_COMPONENT_DRM_OVL_ADAPTOR`
      - `DDP_COMPONENT_MERGE5`
      - `DDP_COMPONENT_DP_INTF1`
  - ddp components are initialized
    - `private->comp_node[i]` are initialized by scanning all sibling OF nodes
    - `private->ddp_comp[i]` are initialized by `mtk_ddp_comp_init`
  - components
    - it seems all component (subdevice) matches are added to
      `component_match`
    - `component_master_add_with_match` waits for all components to match and
      binds the master device
- `mtk_drm_bind` creates a single `drm_device`
  - `mtk_drm_get_all_drm_priv` early returns until `mtk_drm_platform_driver`
    binds to all platform devices
- `mtk_drm_kms_init` initializes the single `drm_device`
  - `mtk_crtc_create` is called for the two pipelines on mt8195
- `drm_dev_register` registers the single `drm_device`
- `drm_fbdev_dma_setup` sets up fbdev emulation

## AFBC

- corsola/cherry advertises AFBC support but triggers
  `mtk-iommu 14016000.iommu: fault type=0x5 iova=0xfec23000 pa=0x0 master=0x51000008(larb=0 port=2) layer=1 read`
  - `PAN_MESA_DEBUG=noafbc` to work around
