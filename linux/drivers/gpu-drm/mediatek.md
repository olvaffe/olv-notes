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
  - `mediatek,mt8196-ovlsys0` for overlay controller 0
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
  - `mediatek,mt8196-ovlsys1` for overlay controller 1
    - this blends 4 planes for external display
    - it registers `clk-mt8196-ovl1` plat dev
    - it registers child nodes as plat devs
      - similar to ovlsys0
    - it registers a `mediatek-drm` plat dev
  - `mediatek,mt8196-dispsys0` for display controller 0
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
  - `mediatek,mt8196-dispsys1` for display controller 1
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
- after `CONFIG_DRM_MEDIATEK` determines the order of the child nodes of the
  controllers, it calls `mtk_ddp_comp_connect` to connect each pair of src/dst
  nodes
  - drm mtk uses fixed paths on older socs or calls
    `mtk_drm_of_ddp_path_build` to build the paths from dt on newer socs
  - `mtk_mmsys_routes` enumerates all supported combinations

## Initialization

- `mtk_drm_init` registers a bunch of platform drivers
  - they will probe plat devs created by `mtk_mmsys_probe`
- `mtk_drm_probe` probes each `mediatek-drm` created by `mtk_mmsys_probe`
  - the parent node is the controller
  - it matches the controller against `mtk_drm_of_ids`
    - before mt8196, the path is hardcoded
    - since mt8196, `mtk_drm_of_ddp_path_build` builds the path from dt
  - `mtk_drm_register_sibling` is called on each child node of the controller
    - `mtk_drm_of_get_ddp_comp_type` returns the comp type of the child node
    - if `MTK_DISP_MUTEX`, `private->mutex_node` is updated
    - if `MTK_DISP_VDISP_AO`, `private->vdisp_ao_node` is updated
    - otherwise,
      - `drm_of_component_match_add` adds a match
      - `mtk_ddp_comp_init` creates a `mtk_ddp_comp` for the child node and
        adds it to `private->hlist`
  - `component_master_add_with_match` registers an aggregate driver
- `mtk_drm_of_ddp_path_build` builds a path from the dt controller
  - there are leader and follower dt controllers
    - this is new since mt8196
    - a controller with `ports` is a leader controller
      - only port 0 as the entrypoints
    - a controller without `ports` but with `direct-link` is a follower
      controller
      - port 0 is async inputs (from another controller)
      - port 1 is async outputs (to another controller)
      - port 2 is input relays (from async inputs)
      - port 3 is output relays (to async outputs)
    - each port can have up to `MAX_CRTC` endpoints
      - `CRTC_MAIN`, `CRTC_EXT`, and `CRTC_THIRD` respectively
    - a crtc consists of a leader controller followed by follower controllers
      - two crtcs can share the same controllers but different endpoints
  - `mtk_drm_of_ddp_path_build_one`
    - `mtk_drm_of_get_ddp_ep_cid` returns `-EREMOTE` if the port connects to
      another controller
    - `out_path->len` is the number of comps (components) in the controller
    - `out_path->order` is the order of the controller in the crtc path
      - it is to be fixed up
- `mtk_drm_bind` is called after all child nodes of a controller are probed
  - `mtk_drm_get_all_drm_priv` returns false unless `mtk_drm_bind` has been
    called on all controllers
    - `private->all_drm_private` are updated to point to each other
  - `drm_dev_alloc` allocs a single `drm_device` containing by all controllers
  - `mtk_drm_kms_init` inits the drm dev
    - `component_bind_all` is called on all controllers
    - `mtk_drm_set_path_orders` updates path orders
      - `path->order` is 0 for the leader controllers
      - it fixes up `path->order` for follower controllers
    - `mtk_crtc_create` creates crtcs
  - `drm_dev_register` registers the drm dev
  - `drm_client_setup` sets up fbdev
- `mtk_crtc_create` is called for each of `mtk_crtc_path`
  - there are 3 paths: `CRTC_MAIN`, `CRTC_EXT`, and `CRTC_THIRD`
  - the caller has a loop to try all controllers
  - `mtk_crtc->mmsys_dev` is the first controller of the crtc
  - `mtk_crtc->ddp_comp` and `mtk_crtc->ddp_comp_nr` are the components
  - `mtk_crtc->vblank_comp_idx` is the first component that generates vblanks
    - e.g., `mediatek,mt8196-disp-outproc`
  - `mtk_crtc->config_comp_idx` is the first component that provides hw planes
    - e.g., `mediatek,mt8196-disp-exdma`
  - `mtk_ddp_comp_register_vblank_cb` registers `mtk_crtc_ddp_irq` as vblank
    irq handler
  - `mtk_crtc->hwlayers` and `mtk_crtc->hwlayer_nr` are the hw layers
    - `mtk_crtc_init_comp_planes` inits hw layers for a component
  - `hwlayer->layer_stages` and `hwlayer->layer_stages_nr` are hw layer stages
    - `mtk_crtc_init_comp_planes` inits stage 0
    - `mtk_crtc_init_layer_stages` inits later stages
      - this is for mt8196+ where a drm plane, aka a hw layer, has >1 components
      - stage 0 is the exdma component
      - stage 1 is the blender component
  - `mtk_crtc_find_suitable_dma_dev` inits `mtk_crtc->dma_dev`
  - `mtk_crtc_init_drm_crtc` inits the drm crtc
  - `mtk_crtc_init_multi_controller_properties` inits multi-controller props
    - a crtc has leader and follower controllers since mt8196
  - `mtk_crtc_init_mutex` inits `mtk_crtc->mutex`
- eDP example
  - `mtk_dp_probe` probes `mediatek,mt8196-edp-tx`
    - it is a bridge that converts internal video signals to edp frames
    - `devm_of_dp_aux_populate_bus` adds the panel from `aux-bus` child node
      - it adds a `dp-aux` dev
      - `dp_aux_ep_probe` matches the `dp-aux` dev with the panel driver
        - `mtk_dp_edp_link_panel`
          - `mtk_dp->next_bridge` points to the panel
          - `devm_drm_bridge_add` adds itself as a bridge
  - `mtk_dvo_probe` probes `mediatek,mt8196-edp-dvo`
    - it is a bridge that converts internal video stream to vidoe signals
    - `mtk_dpi_common_probe` adds itself as a bridge, after the downstream
      bridge is added
  - after successful `mtk_dp_probe` and `mtk_dvo_probe`, `mtk_drm_bind` is called
    - `component_bind_all -> mtk_dpi_common_bind`
      - `drm_encoder_init` inits the drm encoder
      - `drm_bridge_attach` attaches the bridge to the encoder
      - `drm_bridge_connector_init` inits the drm connector

## Atomic Commit

- `drm_mode_atomic_ioctl`
  - it allocs an atomic state to hold user args
  - `prepare_signaling` allocs a out fence state to hold out fances
  - `DRM_MODE_PAGE_FLIP_ASYNC` means non-vsync'ed flip
  - `drm_atomic_nonblocking_commit`
    - `drm_atomic_check_only` validates the atomic state
      - it performs generic checks
      - `config->funcs->atomic_check` is `drm_atomic_helper_check`
        - `drm_atomic_helper_check_modeset`
          - `handle_conflicting_encoders` and `update_connector_routing`
            - `connector_helper_funcs->atomic_best_encoder` is NULL
          - `connector_helper_funcs->atomic_check` is NULL
          - `mode_valid -> mode_valid_path`
            - `encoder_funcs->mode_valid` is NULL
            - `bridge->funcs->mode_valid` is `mtk_dvo_bridge_mode_valid`,
              `mtk_dp_bridge_mode_valid`, etc.
            - `crtc_funcs->mode_valid` is `mtk_crtc_mode_valid`
          - `mode_fixup`
            - `drm_atomic_bridge_chain_check`
              - `bridge->funcs->atomic_check` is
                `mtk_dpi_bridge_atomic_check`, `mtk_dp_bridge_atomic_check`,
                etc.
            - `encoder_helper_funcs->atomic_check` is NULL
            - `crtc_helper_funcs->mode_fixup` is `mtk_crtc_mode_fixup`
        - `drm_atomic_helper_check_planes`
          - `plane_helper_funcs->atomic_check` is `mtk_plane_atomic_async_check`
          - `crtc_helper_funcs->atomic_check` is NULL
    - `config->funcs->atomic_commit` is `drm_atomic_helper_commit`
      - `drm_atomic_helper_setup_commit`
        - `config_helper_funcs->atomic_commit_setup` is NULL
      - `drm_atomic_helper_prepare_planes`
        - `plane_helper_funcs->prepare_fb` is NULL
        - `plane_helper_funcs->begin_fb_access` is NULL
      - if blocking, `drm_atomic_helper_wait_for_fences`
      - `drm_atomic_helper_swap_state`
  - `complete_signaling` returns out fences to userspace
- if non-blocking, `commit_work` calls `commit_tail` from workqueue
  - `drm_atomic_helper_wait_for_fences`
  - `drm_atomic_helper_wait_for_dependencies`
  - `config_helper_funcs->atomic_commit_tail` is
    `drm_atomic_helper_commit_tail_rpm`
    - `drm_atomic_helper_commit_modeset_disables`
      - `disable_outputs`
        - `drm_atomic_helper_commit_encoder_bridge_disable`
          - `bridge->funcs->atomic_disable` is `mtk_dvo_bridge_disable`,
            `mtk_dp_bridge_atomic_disable`, etc.
          - `encoder_helper_funcs->atomic_disable` is NULL
        - `drm_atomic_helper_commit_encoder_bridge_post_disable`
          - `bridge->funcs->atomic_post_disable` is NULL
        - `drm_atomic_helper_commit_crtc_disable`
          - `crtc_helper_funcs->atomic_disable` is `mtk_crtc_atomic_disable`
      - `drm_atomic_helper_commit_crtc_set_mode`
        - `crtc_helper_funcs->mode_set_nofb` is `mtk_crtc_mode_set_nofb`
        - `encoder_helper_funcs->atomic_mode_set` is NULL
        - `bridge->funcs->mode_set` is `mtk_dpi_bridge_mode_set`, etc.
    - `drm_atomic_helper_commit_modeset_enables`
      - `drm_atomic_helper_commit_crtc_enable`
        - `crtc_helper_funcs->atomic_enable` is `mtk_crtc_atomic_enable`
      - `drm_atomic_helper_commit_encoder_bridge_pre_enable`
        - `bridge->funcs->atomic_pre_enable` is NULL
      - `drm_atomic_helper_commit_encoder_bridge_enable`
        - `encoder_helper_funcs->atomic_enable` is NULL
        - `bridge->funcs->atomic_enable` is `mtk_dvo_bridge_enable`,
          `mtk_dp_bridge_atomic_enable`, etc.
    - `drm_atomic_helper_commit_planes`
      - `drm_crtc_helper_funcs->atomic_begin` is `mtk_crtc_atomic_begin`
      - `plane_helper_funcs->atomic_update` is `mtk_plane_atomic_update`
      - `plane_helper_funcs->atomic_enable` is  NULL
      - `plane_helper_funcs->atomic_disable` is `mtk_plane_atomic_disable`
      - `drm_crtc_helper_funcs->atomic_flush` is `mtk_crtc_atomic_flush`
      - `plane_helper_funcs->end_fb_access` is NULL
    - `drm_atomic_helper_fake_vblank`
    - `drm_atomic_helper_commit_hw_done`
    - `drm_atomic_helper_wait_for_vblanks`
    - `drm_atomic_helper_cleanup_planes`
      - `plane_helper_funcs->cleanup_fb` is NULL
- `mtk_crtc_atomic_enable` calls `mtk_crtc_ddp_hw_init`
  - `mtk_mutex_prepare` preps mutex
  - `mtk_crtc_ddp_clk_enable` enables comp clks
  - `mtk_mmsys_default_config` configs mmsys
  - `mtk_ddp_comp_connect` or `mtk_mmsys_hw_connect` connects a pair of comps
  - `mtk_ddp_comp_add` or `mtk_mutex_add_trigger` adds a mutex trigger
  - `mtk_mutex_enable` enables mutex
  - `mtk_ddp_comp_config` configs a comp
  - `mtk_ddp_comp_start` starts a comp
  - `mtk_crtc_config_layer` configs a hwlayer
- `mtk_crtc_atomic_flush` calls `mtk_crtc_update_config`
- upon vblank irq, `mtk_crtc_ddp_irq` is called
  - `mtk_crtc_ddp_config`
  - `mtk_drm_finish_page_flip`

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
