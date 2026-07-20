# Linux DRM Mediatek

## MT8196 DT

- navi internal display
  - data path: `ovlsys0 -> dispsys0 -> dispsys1 -> edp_tx -> panel`
  - `ovlsys0`
    - `ovlsys0_ep_main` is the entrypoint
    - `ovl0_rsz0` sizes the blank stream
    - `ovl0_exdma2 -> ovl0_blender1 -> ... -> ovl0_exdma5 -> ovl0_blender4`
      - each of `ovl0_exdmaX` reads a data plane
      - each of `ovl0_blenderX` blends the plane into the stream
    - `ovl0_outproc0` clamps values, crops stream, and generates vblank irq
    - `direct-link@260` relays `ovl_dl_out_async0_5` to `ovl_dl_out_async0_5`
  - `dispsys0`
    - `direct-link@200` relays `dl_in_async0_0` to `dl_in_relay0_0`
    - `mdp_rsz0` scales the stream
    - `tdshp0` sharpens the stream
    - `ccorr0 -> ccorr1` applies two 3x3 color matrices
    - `gamma0` applies 1d gamma lut
    - `postmask0` blanks out inactive margins
    - `dither0` dithers pixels
    - `direct-link@200` relays `dl_out_relay0_1` to `dl_out_async0_1`
  - `dispsys1`
    - `direct-link@200` relays `dl_in_async1_1` to `dl_in_relay1_1`
    - `dvo0` generates video signal
  - `edp_tx` transcodes the video signal to edp frames
    - `edp_phy` outputs edp signal to `panel`
- navi external display
  - data path: `ovlsys1 -> dispsys1 -> anx_bridge -> usb_c0`
  - `ovlsys1`
    - `ovlsys1_ep_third` is the entrypoint
    - `ovl1_rsz0` sizes the blank stream
    - `ovl1_exdma2 -> ovl1_blender1 -> ... -> ovl1_exdma5 -> ovl1_blender4`
      - each of `ovl1_exdmaX` reads a data plane
      - each of `ovl1_blenderX` blends the plane into the stream
    - `ovl1_outproc0` clamps values, crops stream, and generates vblank irq
    - `direct-link@260` relays `ovl_dl_out_relay1_5` to `ovl_dl_out_async1_5`
  - `dispsys1`
    - `direct-link@200` relays `dl_in_async1_3` to `dl_in_relay1_3`
    - `dsc1` compresses the stream
    - `dsi0` generates video signal
      - `mipi_tx_config0` is the phy
  - `anx_bridge` converts dsi signal to dp signal

## `CONFIG_MTK_MMSYS`

- MMSYS is MediaTek Multimedia Subsystem
  - display
  - camera
  - video
- VMM is Vcore for MMSYS
  - it is a regulator controlled by VCP fw
  - VCP is Video Companion Processor
- `mtk_mmsys_probe` probes
  - `mediatek,mt8196-ovlsys0` for overlay pipeline 0
    - this blends 4 planes for internal display
    - it registers `clk-mt8196-ovl0` plat dev
    - it registers child nodes as plat devs
      - `mediatek,mt8196-ovl-direct-link`
      - `mediatek,mt8196-disp-mutex`
      - `mediatek,mt8196-disp-exdma`
      - `mediatek,mt8196-disp-blender`
      - `mediatek,mt8196-disp-outproc`
      - `mediatek,mt8196-disp-rsz`
    - it registers a `mediatek-drm` plat dev
  - `mediatek,mt8196-ovlsys1` for overlay pipeline 1
    - this blends 4 planes for external display
    - it registers `clk-mt8196-ovl1` plat dev
    - it registers child nodes as plat devs
      - similar to ovlsys0
    - it registers a `mediatek-drm` plat dev
  - `mediatek,mt8196-dispsys0` for display pipeline 0
    - this post-processes display stream for internal display
    - it registers `clk-mt8196-disp0` plat dev
    - it registers child nodes as plat devs
      - `mediatek,mt8196-disp-direct-link`
      - `mediatek,mt8196-disp-mutex`
      - `mediatek,mt8196-disp-aal`
      - `mediatek,mt8196-disp-ccorr`
      - `mediatek,mt8196-disp-dither`
      - `mediatek,mt8196-disp-gamma`
      - `mediatek,mt8196-disp-tdshp`
      - `mediatek,mt8196-disp-postmask`
      - `mediatek,mt8196-disp-rsz`
    - it registers a `mediatek-drm` plat dev
  - `mediatek,mt8196-dispsys1` for display pipeline 1
    - this converts display streams to signals for both internal and external
      displays
    - it registers `clk-mt8196-disp1` plat dev
    - it registers child nodes as plat devs
      - `mediatek,mt8196-disp-direct-link`
      - `mediatek,mt8196-disp-mutex`
      - `mediatek,mt8196-dp-intf`
      - `mediatek,mt8196-disp-dsc`
      - `mediatek,mt8196-dsi`
      - `mediatek,mt8196-edp-dvo`
      - `mediatek,mt8196-disp-dither`
    - it registers a `mediatek-drm` plat dev
  - `mediatek,mt8196-vdisp-ao` for clk
    - it registers `clk-mt8196-vdisp-ao` plat dev
    - no child node
    - it registers a `mediatek-drm` plat dev
- `mediatek-mutex` probes `mediatek,mt8196-disp-mutex`
  - it associates a `mtk_mutex_ctx` with the device

## Initialization

- `mtk_drm_init` registers a bunch of platform drivers
  - they will probe plat devs created by `mtk_mmsys_probe`
- `mtk_drm_probe` probes each `mediatek-drm` created by `mtk_mmsys_probe`
  - the parent node is the pipeline
  - it matches the pipeline against `mtk_drm_of_ids`
    - before mt8196, the path is hardcoded
    - since mt8196, `mtk_drm_of_ddp_path_build` builds the path from dt
  - `mtk_drm_register_sibling` is called on each child node of the pipeline
    - `mtk_drm_of_get_ddp_comp_type` returns the comp type of the child node
    - if `MTK_DISP_MUTEX`, `private->mutex_node` is updated
    - if `MTK_DISP_VDISP_AO`, `private->vdisp_ao_node` is updated
    - otherwise,
      - `drm_of_component_match_add`
      - `mtk_ddp_comp_init`
  - `component_master_add_with_match` registers an aggregate driver
- `mtk_drm_bind` is called after all child nodes of the pipeline are probed
  - `mtk_drm_get_all_drm_priv` returns true when this is the last pipeline
  - `drm_dev_alloc` allocs a `drm_device` shared by all pipelines
  - `mtk_drm_kms_init` inits the drm dev
    - `mtk_crtc_create` creates crtcs
  - `drm_dev_register` registers the drm dev
  - `drm_client_setup` sets up fbdev
- `mtk_drm_platform_driver` binds to the two `mediatek-drm` platform devices on mt8195
  - the compat strings of the mmsys OF nodes are matched again against
    `mtk_drm_of_ids` to get `mtk_mmsys_driver_data`
    - `mt8195_vdosys0_driver_data` has `mt8195_vdo0_legacy_paths`
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
    - `mt8195_vdosys1_driver_data` has `mt8195_vdo1_legacy_paths`
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

## AFBC

- corsola/cherry advertises AFBC support but triggers
  `mtk-iommu 14016000.iommu: fault type=0x5 iova=0xfec23000 pa=0x0 master=0x51000008(larb=0 port=2) layer=1 read`
  - `PAN_MESA_DEBUG=noafbc` to work around

## IOMMU

- device node
  - no `memory-region`
    - otherwise, the driver would call `of_reserved_mem_device_init` to parse
      `memory-region`
    - it would call `rmem->ops->device_init`, which is usually
      - `rmem_dma_device_init` provided by `CONFIG_DMA_DECLARE_COHERENT`
        - `dma_assign_coherent_memory` assigns `dev->dma_mem`
      - `rmem_cma_device_init` provided by `CONFIG_DMA_CMA`
        - it assigns `dev->cma_area`
  - has `iommus`
- platform device
  - `of_platform_default_populate` creates platform devices from dt devices
    - `setup_pdev_dma_masks` sets `pdev->dev.coherent_dma_mask` and
      `pdev->dev.dma_mask` to 32 bits
    - `of_platform_device_create_pdata` overrides `dev->dev.coherent_dma_mask`
      to the same 32 bits
- pre-probe
  - `really_probe` calls `platform_dma_configure`
    - this calls down to `iommu_probe_device` which inits `dev->iommu`,
      `dev->iommu_group`, `dev->dma_iommu`, etc.
    - it is also possible that `bus_iommu_probe` calls `iommu_probe_device` on
      all devices when the controller driver probes
- `dma_alloc_attrs`
  - `get_dma_ops` returns NULL
    - it is always null on x86 and arm64
  - `dev->coherent_dma_mask` is 32-bit unless the driver calls
    `dma_set_coherent_mask` or `dma_set_mask_and_coherent`
  - `dma_alloc_from_dev_coherent` attemps to allocate from `dev->dma_mem`,
    which is null
  - `dma_alloc_direct` returns false because there is `dev->dma_iommu`
    - otherwise, `dma_direct_alloc` would allocate contiguous pages,
      potentially from `dev->cma_area`
  - `iommu_dma_alloc` allocates pages and sets up iommu
    - `__iommu_dma_alloc_noncontiguous`
      - `__iommu_dma_alloc_pages` allocs non-contiguous pages
      - `iommu_dma_alloc_iova` allocs a contiguous iova region
        - this can fail due to the 32-bit limit
      - `sg_alloc_table_from_pages` allocs sgt
      - `iommu_map_sg` maps the non-contiguous pages to the contiguous iova
        region
- `drm_prime_fd_to_handle_ioctl` from `system` dma-heap
  - `drm_gem_prime_fd_to_handle` calls `mtk_gem_prime_import` which calls
    `drm_gem_prime_import_dev`
  - `dma_buf_attach` calls `system_heap_attach` and returns
    `dma_buf_attachment`
  - `dma_buf_map_attachment` calls `system_heap_map_dma_buf` and returns
    `sg_table`
    - `dma_map_sgtable` calls `iommu_dma_map_sg`
      - `iommu_dma_alloc_iova` allocs a contiguous iova region
        - this can fail due to the 32-bit limit
  - `mtk_gem_prime_import_sg_table` creates gem obj from the sgt
