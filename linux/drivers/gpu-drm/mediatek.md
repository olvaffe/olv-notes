Linux DRM Mediatek
==================

## MT8196 DT

- MMSYS is MediaTek Multimedia Subsystem
  - display
  - video codec
- VMM is Vcore for MMSYS
  - it is a regulator controlled by VCP fw
- VCP is Video Companion Processor

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
      `pdev->dev.dma_mask ` to 32 bits
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
