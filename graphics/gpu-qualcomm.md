Qualcomm Adreno
===============

## History

- Snapdragon S series
  - discontinued
  - S1
    - 2007-2008
    - MSM7x27
    - 65nm
    - Adreno 200, GLES 2.0
  - S2
    - 2010
    - MSM7x30, MSM8x55
    - 45nm
    - Adreno 205, GLES 2.0
  - S3
    - 2010
    - MSM8x60
    - 45nm
    - Adreno 220, GLES 2.0
  - S4
    - 2012
    - MSM8x{25,27,30,60}
    - 28nm-45nm
    - Adreno 203, 225, 305, 320
- Snapdragon 2 series
  - ultra budget
  - 200
    - 2013
    - MSM8x{10,12}
    - 28nm
    - Adreno 302
  - 205, 208, 210, 212
    - 2014-2017
    - MSM8x{05,08,09}
    - 28nm
    - Adreno 304
  - 215
    - 2019
    - QM215
    - 28nm
    - Adreno 308
- Snapdragon 4 series
  - entry level
  - 400
    - 2013
    - MSM8x{26,28,30}
    - 28nm
    - Adreno 305
  - 410, 412, 415
    - 2014-2015
    - MSM8x{16,29}
    - 28nm
    - Adreno 306, 405
  - 425, 427, 430, 435
    - 2015-2016
    - MSM8x{17,20,37,40}
    - 28nm
    - Adreno 308, 505
  - 429, 439, 450
    - 2017-2018
    - SDM{429,439,450}
    - 12nm-14nm
    - Adreno 504, 505, 506
  - 460
    - 2020
    - SM4250
    - 11nm
    - Adreno 610
  - 480
    - 2021
    - SM4350
    - 8nm
    - Adreno 619
- Snapdragon 6 series
  - mid-range
  - 600
    - 2013
    - APQ8064
    - 28nm
    - Adreno 320
  - 610, 615, 616
    - 2014-2015
    - MSM89{36,39}
    - 28nm
    - Adreno 405
  - 625, 626
    - 2016
    - MSM8953
    - 14nm
    - Adreno 506
  - 650, 652, 653
    - 2016
    - MSM89{56,76}
    - 28nm
    - Adreno 510
  - 630, 636, 660
    - 2017
    - SDM{630,636,660}
    - 14nm
    - Adreno 508, 509, 512
  - 632, 670
    - 2018
    - SDM{632,670}
    - 10nm-14nm
    - Adreno 506, 615
  - 662, 665, 675, 678
    - 2019-2020
    - SM61{15,25,50}
    - 11nm
    - Adreno 610, 612
  - 680, 690, 695
    - 2020-2021
    - SM{6225,6350,6375}
    - 6nm-8nm
    - Adreno 610, 619
- Snapdragon 7 series
  - upper mid-range
  - 710, 712
    - 2018-2019
    - SDM{710,712}
    - 10nm
    - Adreno 616
  - 720, 730, 732
    - 2019-2020
    - SM71{25,50}
    - 8nm
    - Adreno 618
  - 750, 765, 768
    - 2020
    - SM72{25,50}
    - 7nm
    - Adreno 620
  - 778, 780
    - 2021
    - SM73{25,50}
    - 5nm-6nm
    - Adreno 642
- Snapdragon 8 series
  - high-end
  - 800, 801, 805
    - 2013-2014
    - MSM8x74, APQ8084
    - 28nm
    - Adreno 330, 420
  - 808, 810
    - 2015
    - MSM9x{92,94}
    - 20nm
    - Adreno 418, 430
  - 820, 821
    - 2016
    - MSM8996
    - 14nm
    - Adreno 530
  - 835
    - 2017
    - MSM8998
    - 10nm
    - Adreno 540
  - 845
    - 2018
    - SDM845
    - 10nm
    - Adreno 630
  - 855, 860
    - 2019, 2021
    - SM8150
    - 7nm
    - Adreno 640
  - 865, 870
    - 2020-2021
    - SM8250
    - 7nm
    - Adreno 650
  - 888
    - 2021
    - SM8350
    - 5nm
    - Adreno 660
  - 8 Gen1
    - 2022
    - SM8450
    - 4nm
    - Adreno 730
- Snapdragon 7 Compute Platforms
  - 7c
    - 2020
    - SC7180
    - 8nm
    - Adreno 618
  - 7c Gen2
    - 2021
    - SC7180P
    - 8nm
    - Adreno 618
  - 7c+ Gen3
    - 2021
    - SC7280?
    - 6nm
    - Adreno 7c+ Gen3 (Adreno 635)
- Snapdragon 8 Compute Platforms
  - 835, 850
    - 2018
    - MSM8998, SDM850
    - 10nm
    - Adreno 540, 630
  - 8c, 8cx, 8cx Gen2
    - 2019-2020
    - SC8180
    - 7nm
    - Adreno 675, 680, 690
  - 8cx Gen3
    - 2022
    - SC8???
    - 5nm
    - Adreno 8cx Gen3

## Architecture

- `Qualcomm® Snapdragon™ Mobile Platform OpenCL General Programming and Optimization`
- Adreno
  - a big L2 sitting in from of system memory
    - all memory accesses from SP are via L2
  - a Texture Processor / L1 for Sampling and Image Read
  - multiple Shader Processors
- a Shader Processor (SP) is
  - core block of Adreno GPUs with many moudles including ALU, load/store
    unit, control flow unit, register files, etc.
  - each SP corresponds to a OpenCL compute unit
  - load/store through L2 for buffer objects and (r/w) image objects
  - load/sample through  texture processor / L1 for read-ony image objects
- a Texture Processor (TP) is
  - texture fetching and filtering
  - coupled with L1 which fetches data from L2
- A unified L2 cache (UCHE)
  - respond to SP's load/store and L1's load requests
- Waves and fibers
  - the smallest unit of execution is a fiber (thread...)
  - a collection of fibers execute in lock-step is a wave
  - wave size can be 8, 16, 32, 64, 128, etc.
  - a SP can have multiple active waves, for latency hiding
    - maximum number of active waves depend on register file size and register
      footprint of the waves
  - an OpenCL kernel launches multiple workgroups
    - each workgroup is assigned to an SP
    - each SP processes one (or multiple on highend) workgroup at a time
    - the remaining SPs are queued in GPU for execution
  - an OpenCL workgroup is processed by multiple waves
    - the larger the better, but not always
    - larger workgroups means more waves and better latency hiding
- OpenCL Memory Model
  - global memory is system memory
  - local memory is on-chip GMEM in SP
  - private memory is registers, on-chip GMEM, or system memory decided by
    compiler
  - constant memory is on-chip if can fit; system memory otherwise
  - not coherent between CPU/GPU on A5xx
- Graphics Fixed-Function Pipeline
  - Command Processor (CP)
    - uncached
  - Color Cache Unit (CCU)
    - draws and blits hit CCU

## Device Identification

- device tree specifies `qcom,adreno-635.0`
  - msm parses `635.0` into `core`, `major`, `minor`, and `patch`
  - msm then uses the four numbers to look up the `adreno_info`
  - `MSM_PARAM_CHIP_ID` returns the 4 versions plus "speed bin" packed into
    64-bit
    - speed bin determines the max frequency
  - `MSM_PARAM_GPU_ID` returns 0
    - it is deprecated and can return numbers such as `618` on older gpus
- userspace uses chip id to look up the `fd_dev_info`
  - 635 vs 618
    - `fibers_per_sp` is `128 * 2 * 16`, not `128 * 16`
    - `reg_size_vec4` is 64, not 96
    - `instr_cache_size` is 127, not 64
    - `TPL1_DBG_ECO_CNTL` is `0x5008000`, not `0x100000`
    - these flags are set on 635
      - `supports_multiview_mask`
      - `has_z24uint_s8uint`
      - `tess_use_shared`
      - `storage_16bit`
        - `VK_KHR_16bit_storage`
      - `has_tex_filter_cubic`
        - `VK_EXT_filter_cubic`
      - `has_sample_locations`
        - `VK_EXT_sample_locations`
      - `has_lpac`
      - `has_shading_rate`
      - `has_getfiberid`
        - more `subgroupSupportedStages`
      - `has_dp2acc`
        - `integerDotProduct4x8BitPackedUnsignedAccelerated` and more
      - `has_dp4acc`
    - these flags are NOT set on 635
      - `has_cp_reg_write`
      - `has_8bpp_ubwc`
      - `ccu_cntl_gmem_unk2`
      - `indirect_draw_wfm_quirk`
      - `depth_bounds_require_depth_test_quirk`
    - `num_sp_cores` is 2, not 1
    - `num_ccu` is 2, not 1
    - `RB_UNKNOWN_8E04_blit` is both `0x00100000`
    - `PC_POWER_CNTL` is 1, not 0

## Overview

- relevant nodes in `sc7180-trogdor-coachz-r3.dts`
  - `simple-bus`
    - `qcom,sc7180-mdss`
      - MSM Mobile Display Subsystem (MDSS) that encapsulates sub-blocks like
        DPU display controller, DSI and DP interfaces etc.
      - `qcom,dsi-phy-10nm`
      - `qcom,sc7180-dpu`
      - `qcom,mdss-dsi-ctrl`
      - `qcom,sc7180-dp`
    - `qcom,adreno`
    - `qcom,geni-se-qup`
      - Generic Interface (GENI) Serial Engine (SE) Wrapper driver for Qualcomm
        Universal Peripheral (QUP) Wrapper controller, designed to support
        various serial bus protocols like UART, SPI, I2C, I3C, etc.
      - `qcom,geni-i2c`
        - `ti,sn65dsi86`
          - DSI-to-eDP bridge
          - `boe,nv110wtm-n61`
  - `pwm-backlight`
- `msm_drm_register` registers these platform drivers in order
  - unused `mdp5_driver` named `msm_mdp` for `qcom,mdp5` and more
  - `dpu_driver` named `msm_dpu` for `qcom,sc7180-dpu` and more
  - `dsi_phy_platform_driver` named `msm_dsi_phy` for `qcom,dsi-phy-10nm` and
    more
  - `dsi_driver` named `msm_dsi` for `qcom,mdss-dsi-ctrl` and more
  - unused `msm_hdmi_phy_platform_driver` named `msm_hdmi_phy` for
    `qcom,hdmi-phy-8996` and more
  - unused `msm_hdmi_driver` named `hdmi_msm` for `qcom,hdmi-tx-8996` and more
  - `dp_display_driver` named `msm-dp-display` for `qcom,sc7180-dp` and more
  - `adreno_driver` named `adreno` for `qcom,adreno` and more
  - `msm_platform_driver` named `msm` for `qcom,sc7180-mdss` and more
- `adreno_probe` probes `qcom,adreno`
  - this only calls `component_add`
- `msm_pdev_probe` probes `qcom,sc7180-mdss`
  - this calls `component_match_add` to match all relevant subdevices
  - this calls `component_master_add_with_match` to register the aggregate
    driver
- `msm_drm_bind`
  - this is called after all components have been successfully bound
    - `adreno_bind`
    - `dpu_bind`
    - `dp_display_bind`
    - `dsi_bind`
 
## devfreq

- msm internally has
  - `msm_gpu_devfreq::boost_freq` and `msm_gpu_devfreq::idle_freq`
    - `boost_freq` is a `DEV_PM_QOS_MIN_FREQUENCY` request
      - initialized to 0
    - `idle_freq` is a `DEV_PM_QOS_MAX_FREQUENCY` request
      - initialized to default, which is `S32_MAX`
  - `msm_devfreq_active` is called when the device is idle and a new submit
    is scheduled
    - if we've been in idle for a while (50ms)
      - update `boost_freq` to double the current freq
      - schedule `msm_devfreq_boost_work` to reset `boost_freq` to 0
    - update `idle_freq` to default
  - `msm_devfreq_idle` is called when all scheduled submits complete
    - schedule `msm_devfreq_idle_work`, which update idle timestamp
      - on supported devices (618 only for now), it also sets `idle_freq` to 0
- msm adds a devfreq device with `simple_ondemand`
  - the governor calls `devfreq_monitor_start` to start polling
  - every 50ms, `devfreq_update_target` asks the governor for the target freq,
    clamp it to `[DEV_PM_QOS_MIN_FREQUENCY, DEV_PM_QOS_MAX_FREQUENCY]`
  - `devfreq_set_target` sets and returns the effective freq

## GMEM

- kernel
  - `MSM_PARAM_GMEM_BASE` is `0x100000` mostly
  - `MSM_PARAM_GMEM_SIZE` is `SZ_512K` on a618/a635
- Mesa `struct fd_dev_info`
  - `tile_align_w` and `tile_align_h`
    - alignments of the dimensions of a tile
    - 32x32 mostly
  - `gmem_align_w` and `gmem_align_h`
    - what `vkGetRenderAreaGranularity` returns
    - and what?
    - 16x4 mostly
  - `tile_max_w` and `tile_max_h`
    - max size of a tile
    - 1024x1008 mostly
  - `num_ccu` is 1 on a618 and 2 on a635
- Mesa `tu_physical_device`
  - `ccu_offset_bypass` reserves 64KB per CCU from gmem as depth cache, used
    in sysmem mode
  - `ccu_offset_gmem` reserves 16KB per CCU from gmem for msaa resolves, used
    in gmem mode
- Mesa `tu_render_pass_gmem_config`
  - `gmem_align` is 8KB mostly
  - the goal is to decide the number of gmem blocks allocated for each
    attachment
    - `ccu_offset_gmem` is 496KB mostly, thus `gmem_blocks` is 62
    - let's say att 1 has cpp 4 and att 2 has cpp 2
    - it allocates `62*4/6` to att 1 and `62*2/6` to att 2
      - that is, it allocates 41 blocks to att 1 and 21 blocks to att 2
      - ot, it allocates `41*8192/4` and `21*8192/2` pixels for att 1 and 2
      	respectively
    - `pass->gmem_pixels` is set to the max numbers of pixels that can fit
      gmem for this render pass
- Mesa `tu_tiling_config_update_tile_layout`
  - compute `fb->tile_count`, number of tiles in each direction
  - compute `fb->tile0`, dimensions of a tile
  - they must obey hw limits that are already computed and saved in render
    pass
- Mesa `tu_tiling_config_update_pipe_layout`
  - compute `fb->tile_count`, number of tiles in each direction per pipe
  - there is a total of 32 pipes; we want to use as many pipes as possible

## MDSS

- componentized device
  - when a subdevice is probed, add it with "component_add"
  - when the master device is probed, describe the required subdevices and
    add the master with "component_master_add_with_match"
    - it will attempt to bind the master device to its driver
    - the driver init function calls "component_bind_all" to bind the
      subdevices to their respective drivers
- "msm_drm_init" is the driver init function for the master device
  - it initializes MDSS with "dpu_mdss_init" first
  - it then binds all subdevices
  - the binding of DPU initializes KMS with "dpu_bind"
  - it then calls kms's "hw_init"
  - it starts a "crtc_commit:%d" and a "crtc_event:%d" thread for each crtc
  - "drm_vblank_init"

## DRM funcs

- static const struct drm_mode_config_funcs mode_config_funcs
       .fb_create = msm_framebuffer_create,
       .output_poll_changed = drm_fb_helper_output_poll_changed,
       .atomic_check = drm_atomic_helper_check,
       .atomic_commit = drm_atomic_helper_commit,
- static const struct drm_mode_config_helper_funcs mode_config_helper_funcs
       .atomic_commit_tail = msm_atomic_commit_tail,
- static const struct drm_framebuffer_funcs msm_framebuffer_funcs
       .create_handle = drm_gem_fb_create_handle,
       .destroy = drm_gem_fb_destroy,
- static const struct drm_connector_funcs dsi_mgr_connector_funcs
       .detect = dsi_mgr_connector_detect,
       .fill_modes = drm_helper_probe_single_connector_modes,
       .destroy = dsi_mgr_connector_destroy,
       .reset = drm_atomic_helper_connector_reset,
       .atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,    
       .atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
- static const struct drm_connector_helper_funcs
       .get_modes = dsi_mgr_connector_get_modes,
       .mode_valid = dsi_mgr_connector_mode_valid,
       .best_encoder = dsi_mgr_connector_best_encoder,
- static const struct drm_bridge_funcs dsi_mgr_bridge_funcs
       .pre_enable = dsi_mgr_bridge_pre_enable,
       .enable = dsi_mgr_bridge_enable,
       .disable = dsi_mgr_bridge_disable,
       .post_disable = dsi_mgr_bridge_post_disable,
       .mode_set = dsi_mgr_bridge_mode_set,
- static const struct drm_encoder_funcs dpu_encoder_funcs
       .destroy = dpu_encoder_destroy,
       .late_register = dpu_encoder_late_register,
       .early_unregister = dpu_encoder_early_unregister, 
- static const struct drm_encoder_helper_funcs
       .mode_set = dpu_encoder_virt_mode_set,
       .disable = dpu_encoder_virt_disable,
       .enable = dpu_kms_encoder_enable,
       .atomic_check = dpu_encoder_virt_atomic_check,
       .commit = dpu_encoder_virt_enable,
- static const struct drm_plane_funcs dpu_plane_funcs
       .update_plane = drm_atomic_helper_update_plane,
       .disable_plane = drm_atomic_helper_disable_plane,
       .destroy = dpu_plane_destroy,
       .reset = dpu_plane_reset,
       .atomic_duplicate_state = dpu_plane_duplicate_state,
       .atomic_destroy_state = dpu_plane_destroy_state,
       .late_register = dpu_plane_late_register,
       .early_unregister = dpu_plane_early_unregister,
- static const struct drm_plane_helper_funcs dpu_plane_helper_funcs
       .prepare_fb = dpu_plane_prepare_fb,
       .cleanup_fb = dpu_plane_cleanup_fb,
       .atomic_check = dpu_plane_atomic_check,
       .atomic_update = dpu_plane_atomic_update,
- static const struct drm_crtc_funcs dpu_crtc_funcs
       .set_config = drm_atomic_helper_set_config,
       .destroy = dpu_crtc_destroy,
       .page_flip = drm_atomic_helper_page_flip,
       .reset = dpu_crtc_reset,
       .atomic_duplicate_state = dpu_crtc_duplicate_state,       
       .atomic_destroy_state = dpu_crtc_destroy_state, 
       .late_register = dpu_crtc_late_register,
       .early_unregister = dpu_crtc_early_unregister,
- static const struct drm_crtc_helper_funcs dpu_crtc_helper_funcs
       .disable = dpu_crtc_disable,
       .atomic_enable = dpu_crtc_enable,
       .atomic_check = dpu_crtc_atomic_check,
       .atomic_begin = dpu_crtc_atomic_begin,
       .atomic_flush = dpu_crtc_atomic_flush,
- static const struct file_operations fops
       .owner              = THIS_MODULE,
       .open               = drm_open,
       .release            = drm_release,
       .unlocked_ioctl     = drm_ioctl,
       .compat_ioctl       = drm_compat_ioctl,
       .poll               = drm_poll,
       .read               = drm_read,
       .llseek             = no_llseek,
       .mmap               = msm_gem_mmap,
- static const struct vm_operations_struct vm_ops
       .fault = msm_gem_fault,
       .open = drm_gem_vm_open,
       .close = drm_gem_vm_close,
- static const struct drm_driver msm_driver
       .driver_features    = DRIVER_HAVE_IRQ |
                               DRIVER_GEM |
                               DRIVER_PRIME |
                               DRIVER_RENDER |
                               DRIVER_ATOMIC |
                               DRIVER_MODESET,
       .open               = msm_open,
       .postclose           = msm_postclose,
       .lastclose          = drm_fb_helper_lastclose,
       .irq_handler        = msm_irq,
       .irq_preinstall     = msm_irq_preinstall,
       .irq_postinstall    = msm_irq_postinstall,
       .irq_uninstall      = msm_irq_uninstall,
       .enable_vblank      = msm_enable_vblank,
       .disable_vblank     = msm_disable_vblank,
       .gem_free_object    = msm_gem_free_object,
       .gem_vm_ops         = &vm_ops,
       .dumb_create        = msm_gem_dumb_create,
       .dumb_map_offset    = msm_gem_dumb_map_offset,
       .prime_handle_to_fd = drm_gem_prime_handle_to_fd,
       .prime_fd_to_handle = drm_gem_prime_fd_to_handle,
       .gem_prime_export   = drm_gem_prime_export,
       .gem_prime_import   = drm_gem_prime_import,
       .gem_prime_res_obj  = msm_gem_prime_res_obj,
       .gem_prime_pin      = msm_gem_prime_pin,
       .gem_prime_unpin    = msm_gem_prime_unpin,
       .gem_prime_get_sg_table = msm_gem_prime_get_sg_table,
       .gem_prime_import_sg_table = msm_gem_prime_import_sg_table,
       .gem_prime_vmap     = msm_gem_prime_vmap,
       .gem_prime_vunmap   = msm_gem_prime_vunmap,
       .gem_prime_mmap     = msm_gem_prime_mmap,
       .debugfs_init       = msm_debugfs_init,
       .ioctls             = msm_ioctls,
       .num_ioctls         = ARRAY_SIZE(msm_ioctls),
       .fops               = &fops,
       .name               = "msm",
       .desc               = "MSM Snapdragon DRM",
       .date               = "20130625",
       .major              = MSM_VERSION_MAJOR,
       .minor              = MSM_VERSION_MINOR,
       .patchlevel         = MSM_VERSION_PATCHLEVEL,

## DPU

## Memory Management

- ioctls
  - MSM_GEM_NEW
    - msm_ioctl_gem_new takes a size and a flags
       - it kzallocs a msm_gem_object, which is also a drm_gem_object
       - the gem object is added to the device's private inactive_list
       - drm_gem_object_init is called to set up filp
       - drm_gem_handle_create is called and the handle is the id into file's private object_idr
  - DRM_IOCTL_GEM_CLOSE
    - drm_gem_handle_delete is called and it deletes the handle in file's private object_idr
    - drm_gem_object_release_handle is called
      - if there are prime handles, they are removed from file's private prime table
      - if there are vmas with this file's tag, they are removed
      - if there is a name, it is removed from the device's object_name_idr
      - if exported as dma_buf, the handle is removed
  - DRM_IOCTL_GEM_FLINK
    - it creates a name for the BO and adds the name to device's object_name_idr
  - DRM_IOCTL_GEM_OPEN
    - it looks up a BO using its name, and create a handle to the file's object_idr
  - DRM_IOCTL_PRIME_HANDLE_TO_FD
    - it looks up a BO from the file's object_idr
    - it calls drm_gem_prime_export and save the dma_buf to BO's dma_buf
    - it inserts the dma_buf into file's prime table
    - it finds an fd for the dma_buf and return the fd
    - if the BO was imported, use the original dma_buf
    - if the BO has been exported, use the same dma_buf
  - DRM_IOCTL_PRIME_FD_TO_HANDLE
    - if the dma-buf was already imported or exported, return
    - if the dma-buf was created by the device, grab the backing BO
    - if the dma-buf points to a non-GEM BO, create a new BO around its pages
    - create a handle for the BO in file's object_idr
    - add the dma_buf to the file's prime table
  - MSM_GEM_INFO
    - when MSM_INFO_IOVA is set, return the offset in GPU address_space
      - create a new VMA
      - get_pages() to get the pages and sg table
      - msm_gem_map_vma to initialize the VMA and to set up IOMMU
      - return the offset of the VMA
    - otherwise, intialize obj->vma_node within dev->vma_offset_manager and
      returns the VMA offset
  - MSM_GEM_MADVISE
  - MSM_GEM_CPU_PREP
  - MSM_GEM_CPU_FINI
- mmap
  - use the VMA to look up the object from dev->vma_offset_manager
  - drm_vma_node_is_allowed checks if the object has an associlated file private handle
  - set vma->vm_ops to driver's vm_operations_struct
  - in fault handler, get_pages() and update MMU
- reservation_object
  - each newly created BO owns a reservation_object
  - each dma-buf import creates a BO with the reservation_object of the dma-buf
  - a reservation_object can hold an exclusive dma_fence and a list of shared dma_fence

## Command Submission and Fence

- Fence
  - msm_fence_context is a seqno-based per-ring fence context
    - it goes by the name "gpu-ring-%d"
    - each dma_fence allocated from the context is assocaited with a seqno (++fctx->last_fence)
    - each bo submission is assocaited with a dma_fence (and its seqno)
    - when a submission completes, msm_update_fence is called to update
      fctx->completed_fence to the submission's seqno.
    - to check if a dma_fence is signaled, (fctx->completed_fence >= fence->seqno)
  - each submission may have a fence fd input
    - it must point a valid sync_file if not -1
    - sync_file_get_fence converts it to a dma_fence
    - if the dma_fence is not from ring->fctx, do a blocking dma_fence_wait
  - each submission may request a fence fd output
    - it converts the dma_fence associated with the submission into a sync_file
    - and return the fd to the userspace
- MSM_GEM_SUBMIT
  - a submission consists of a bunch of drm_msm_gem_submit_cmd and drm_msm_gem_submit_bo
    - each bo has a flag, indicating
  - submit_lookup_objects initializes submit->bos from drm_msm_gem_submit_bo array
    - it looks up handles from file->object_idr to get BOs
    - it temporarily sets iova of each BOs to the presumed address set by userspace
  - submit_lock_objects locks the BOs' reservation_object
    - if failed to lock any BO, it does a slow lock on the BO
  - submit_fence_sync waits for the BOs to be ready
    - if the BO is only read-only by the submission, reserve a shared fence
      for the BO.  The shared fence makes sure no one writes to the BO before
      the submission completes.
    - return early if the explicit fence fd input is used
    - otherwise, msm_gem_sync_object is called
      - if there is a explicit fence for the BO (someone wrote to the BO), do
        a CPU wait with dma_fence_wait
      - if we are going to write to the BO, do CPU wait on all shared fences
        of the BO
  - submit_pin_objects pins all BOs and finalizes their IOVAs
    - if presumed address is correct, mark BO_VALID
    - otherwise, clear BO_VALID.  Need to apply reloc.
  - for each drm_msm_gem_submit_cmd,
    - lookup the BO (which is specified using drm_msm_gem_submit_bo)
    - remember the type, offseted IOVA, size
    - if reloc is required, submit_reloc applies the reloc list
  - msm_gpu_submit is called
    - it calls msm_gpu_hw_init in case the GPU was suspended
    - "submit->seqno = ++ring->seqno", which the ring's seqno, unrelated to fence seqno
    - it dumps the submission with msm_rd_dump_submit, if the debugfs file is opened
    - it updates GPU SW counters
    - for each BO, if marked for reading or writing, call msm_gem_move_to_active
      - this adds the submission's dma_fence to the reservation_object of the BO
      - also move the BO from drm inactive_list to gpu active_list
    - a6xx_submit is called
      - for each cmd,
        - emit CP_INDIRECT_BUFFER_PFE with the cmd's IOVA offset and size
      - emit REG_A6XX_CP_SCRATCH_REG to write submit->seqno (ring seqno) to scratch reg
      - emit CP_EVENT_WRITE
        - arg1: request timestamp and an interrupt
        - arg2: address of msm_rbmemptrs.fence
        - arg3: submit->seqno
        - this writes submit->seqno to msm_rbmemptrs.fence
      - write to REG_A6XX_CP_RB_WPTR to update write pointer
    - there are three types of commands
      - MSM_SUBMIT_CMD_IB_TARGET_BUF: emit CP_INDIRECT_BUFFER_PFE to execute
      - MSM_SUBMIT_CMD_CTX_RESTORE_BUF: emit CP_INDIRECT_BUFFER_PFE to execute
        _if_ the last submit is from a different opened drm file
      - MSM_SUBMIT_CMD_IB_TARGET_BUF: ignored (except reloc was applied in a prior step)
  - msm_gpu_retire is triggered by the irq requested by msm_gpu_submit 
    - it wakes up retire_worker
      - update_fences calls msm_update_fence with all completed submission fences
      - for each completed submission, its msm_gem_submit is freed
    - it updates SW counters
- other ioctls
  - MSM_SUBMITQUEUE_NEW
    - create a file-local msm_submitqueue for a ring buffer
  - MSM_SUBMITQUEUE_CLOSE
    - destroy a msm_submitqueue
  - MSM_WAIT_FENCE is legacy path and calls msm_wait_fence to wait for a specified seqno

## GPU

- When device "qcom,adreno" is bound,
  - adreno_info is determined
        {
               .rev = ADRENO_REV(6, 3, 0, ANY_ID),
               .revn = 630,
               .name = "A630",
               .fw = {
                       [ADRENO_FW_SQE] = "a630_sqe.fw",
                       [ADRENO_FW_GMU] = "a630_gmu.bin",
               },
               .gmem = SZ_1M,
               .inactive_period = DRM_MSM_INACTIVE_PERIOD,
               .init = a6xx_gpu_init,
       },
  - init is called to get an "msm_gpu"
    - a6xx_gpu_init calls adreno_gpu_init with 1 ring
    - msm_gpu_init creates a 4G address space as requested
    - it also creates 1 msm_ringbuffer as requested
      - a 32K BO
      - start/end/enxt/cur as indices
      - submits is a list of ?
      - fctx is a fence context goes by the name "gpu-ring-%d"
      - memptrs points to a BO managed by msm_gpu
  - static const struct adreno_gpu_funcs funcs
       .base = {
               .get_param = adreno_get_param,
               .hw_init = a6xx_hw_init,
               .pm_suspend = a6xx_pm_suspend,
               .pm_resume = a6xx_pm_resume,
               .recover = a6xx_recover,
               .submit = a6xx_submit,
               .flush = a6xx_flush,
               .active_ring = a6xx_active_ring,
               .irq = a6xx_irq,
               .destroy = a6xx_destroy,
               .show = a6xx_show,
               .gpu_busy = a6xx_gpu_busy,
               .gpu_get_freq = a6xx_gmu_get_freq,
               .gpu_set_freq = a6xx_gmu_set_freq,
       },
       .get_timestamp = a6xx_get_timestamp,
- when the DRM device is opened, adreno_load_gpu is called
  - it loads the GPU firmware
  - turns on the PM
  - calls msm_gpu_hw_init

## debugfs

- dma-buf
  - /d/dma_buf/buf_info
    - "dma_buf_export" creates a new dma_buf.  This file shows all dma_buf,
      who did the exports, exclusive and shared fences,
- drm_core_init adds
  - /d/dri/drm_master_relax allows normal users to set/drop master
- drm_debugfs_init adds
  - /d/dri/<minor>/name dumps dev name
  - /d/dri/<minor>/clients dumps all opened files of the dev
  - /d/dri/<minor>/gem_names dumps object_name_idr (flink names)
- drm_atomic_debugfs_init adds
  - /d/dri/<minor>/state, enabled when atomic commit is supported, print the plane/crtc/connector states
- drm_framebuffer_debugfs_init adds
  - /d/dri/<minor>/framebuffer lists the current framebuffers
- drm_client_debugfs_init adds
  - /d/dri/<minor>/internal_clients lists the in-kernel clients, which is only used by fbdev
- drm_debugfs_crtc_add adds
  - /d/dri/<minor>/crtc-<index>
- drm_debugfs_connector_add adds
  - /d/dri/<minor>/<connector-name>/force to force connector on/off
  - /d/dri/<minor>/<connector-name>/edid_override for EDID override
- msm_debugfs_init adds
  - /d/dri/<minor>/gem dumps all GEM objects
  - /d/dri/<minor>/mm dumps vma_offset_manager mm
  - /d/dri/<minor>/fb dumps all framebuffers
  - /d/dri/<minor>/gpu dumps GPU states (crash the system!)
- dpu_kms_debugfs_init adds
  - /d/dri/<minor>/debug/*, a bunch of stuff
- msm_rd_debugfs_init adds
  - /d/dri/<minor>/rd, when opened, dump all submits
  - /d/dri/<minor>/hangrd, when opened, dump the submit that triggers hangchecker
- msm_perf_debugfs_init adds
  - /d/dri/<minor>/perf, when opened, starts SW and hW counters and dumps the stats
- _dpu_crtc_init_debugfs adds
  - /d/dri/<minor>/crtc<id>
- _dpu_encoder_init_debugfs adds
  - /d/dri/<minor>/encoder<id>
- _dpu_plane_init_debugfs adds
  - /d/dri/<minor>/plane<id>
   
## cffdump

- rd_open writes RD_GPU_ID
- msm_rd_dump_submit writes
  - RD_CMD for comm name, comm pid, submit ring seqno
  - if module loaded with rd_full=true, for each BO,
    - write a RD_GPUADDR
    - write a RD_BUFFER_CONTENTS
  - for each cmd,
    - write a RD_GPUADDR for the BO
    - write a RD_BUFFER_CONTENTS for the BO
    - write a RD_CMDSTREAM_ADDR

## PM4 Command Packets

- From Radeon South Island Programming Guide
  - Packet DW0[31:30] spcifies the type
  - Type 0 updates registers in the first 64K DW
    - use type 3 instead
    - type0(u16 startReg, u14 count, u32 value[count])
  - Type 2 is no-op, to fill up the trailing space
  - Type 3 executes the specified opcode
    - tyep3(u8 opcode, bool predicate, u32 payload[])
    - Initialization Packets
      - MI_INITIALIZE
    - Command Buffer Packets
      - INDIRECT_BUFFER
    - Draw/Dispatch Packets
      - DRAW_INDEX
      - DRAW_INDEX_AUTO
      - DRAW_INDIRECT
    - State Management Packets
    - Command Predication Packets
    - Synchronization Packets
      - EVENT_WRITE: initiate an event and optionally write a value to the
        specified address when the event completes
    - Atomic
    - Misc Packets
- From freedreno
  - type 7 works similar to type 3
  - type 4 works similar to type 0

## Freedreno

- fd_draw_vbo emits into batch->draw using fd6_draw_vbo
  - fd6_emit_state
    - check dirty states and create (or reference pre-created) state objects (small IBs)
      - FD6_GROUP_VBO
      - FD6_GROUP_VBO_BINNING
      - FD6_GROUP_ZSA
      - FD6_GROUP_LRZ
      - FD6_GROUP_LRZ_BINNING
      - FD6_GROUP_PROG
      - FD6_GROUP_PROG_BINNING
      - FD6_GROUP_RASTERIZER
      - FD6_GROUP_VS_CONST
      - FD6_GROUP_VS_TEX
      - FD6_GROUP_FS_TEX
    - emit many other non-stateobj states
    - SET_DRAW_STATE() to reference the stateobjs
      - the opcode specifies a set of IBs and a set of masks saying when are the IBs applied (binning or rendering)
        - the IBs are probably cached in an internal memory
        - so that HW can execute different IBs in different states (binding or rendering)
      - each IB has a key to allow partial updates
        - update this IB in the internal cache but not others
  - REG_WRITE(VFD_INDEX_OFFSET): first index
  - REG_WRITE(PC_RESTART_INDEX): primitive restart
  - emit_marker6(): for debugging
  - DRAW_INDEX_OFFSET(draw_patches, instance_count, count)
  - emit_marker6(): for debugging
- fd_batch_flush queues jobs into "flush_queue" thread
  - fd_gmem_render_tiles does the jobs
    - sysmem mode: bypassing GMEM and is used for very simple operations (blits)
      - emit_sysmem_prep
      - emit_ib
      - emit_sysmem_fini
    - gmem mode:
      - emit_tile_init
      - for each tile
        - emit_tile_prep
        - emit_tile_mem2gmem
        - emit_tile_renderprep
        - emit_ib
        - emit_tile_gmem2mem
      - emit_tile_fini
- pipeline
  - VFD: vertex fetch and decode
  - SP: stream processor
  - VPC: to store vs outout
  - GRAS: rasterizer
  - RB: render buffer?
  - RBBM
  - CP
  - PC
  - HLSQ: shader resoureces?
  - VBIF
  - VSC
- VFD (vertex fetch and decode)
  - set_vertex_buffer
    - REG_WRITE(VFD_FETCH) and friends in FD6_GROUP_VBO
  - bind_vertex_element_states
    - REG_WRITE(VFD_DECODE) and friends in FD6_GROUP_VBO
- VS
  - set_clip_state
    - baked into VS
  - set_constant_buffer
    - CP_LOAD_STATE6_GEOM(offset=slot, SB6_VS_SHADER, ST6_CONSTANTS) in FD6_GROUP_VS_CONST
  - set_shader_buffers
    - CP_LOAD_STATE6_GEOM(SB6_SSBO) in draw IB
    - CP_LOAD_STATE6_GEOM(SB6_VS_SHADER) in draw IB, to describe buffer sizes
  - bind_sampler_states and set_sampler_views
    - save to 'ctx->tex[shaderStage]' first and process later in fd6_emit_textures
    - convert sampler states to an array of (4-dword SAMPLER_STATE) in BO
    - convert sampler views to an array of (16-dword SAMPLER_VIEW) in BO
    - in FD6_GROUP_VS_TEX IB,
      - CP_LOAD_STATE6_GEOM(SB6_VS_TEX) to load both arrays
      - REG_WRITE(P_VS_TEX_SAMP_LO) and friends
  - bind_vs_state
    - in creation, NIR IR is stored
      - HW instr is not generated until the "variant" is known
    - CP_LOAD_STATE6_GEMO(offset=0, SB6_VS_SHADER, ST6_SHADER, indirect) to load the consts
    - fd6_program_create looks at all bs (binning shader), vs, and fs to create a stateobj
      - REG_WRITE(SP_VS_CONFIG)
      - REG_WRITE(HLSQ_VS_CNTL)
      - REG_WRITE(SP_VS_CTRL_REG0)
      - REG_WRITE(VPC_VAR_DISABLE)
      - REG_WRITE(SP_VS_OUT_REG)
      - REG_WRITE(SP_VS_VPC_DST_REG)
      - REG_WRITE(SP_VS_OBJ_START_LO)
      - CP_LOAD_STATE6_GEOM() to load VS
      - REG_WRITE(SP_PRIMITIVE_CNTL)
- GRAS (gemoetry rasterizer?)
  - set_viewport_states
    - REG_WRITE(GRAS_CL_VPORT_XOFFSET_0) and friends in draw IB
  - bind_rasterizer_state
    - in state creation, an IB is created to REG_WRITE(GRAS_SU_CNTL) and friends
    - in bind, the IB is used as FD6_GROUP_RASTERIZER
    - the state also interacts with other states
  - set_scissor_states
    - REG_WRITE(GRAS_SC_WINDOW_SCISSOR_TL) and friends in gmem IB
    - affects binning
- VSC (binning / visibility stream c?)
- FS
  - see VS
  - set_shader_images
    - CP_LOAD_STATE6_FRAG(offset=slot, SB6_FS_TEX, ST6_CONSTANTS, direct) for use by imageLoad
    - CP_LOAD_STATE6_FRAG(offset=slot, SB6_SSBO, ST6_2, direct) for use by imageStore
    - CP_LOAD_STATE6_FRAG(SB6_FS_SHADER) to describe image dimensions
- RB (render buffer?)
  - set_blend_color
    - REG_WRITE(RB_BLEND_RED_F32) and friends in draw IB
  - set_stencil_ref
    - REG_WRITE(RB_STENCILREF) in draw IB
  - bind_blend_states
    - REG_WRITE(RB_MRT_CONTROL) and friends in draw IB
  - bind_depth_stencil_alpha_state
    - in state creation, an IB is created to REG_WRITE(RB_ALPHA_CONTROL) and friends
    - in bind, the IB is used as FD6_GROUP_ZSA
  - set_framebuffer_state

## Freedreno render_tiles

- fd6_emit_tile_init
  - fd6_emit_restore
    - fd6_cache_flush
      - EVENT_WRITE(0x31) to trigger cache flush
    - REG_WRITE(HLSQ_UPDATE_CNTL, 0xfffff)
    - REG_WRITE(a bunch of registers)
    - emit_marker6
      - REG_WRITE(SCRATCH_REG7, ++marker_cnt)
    - REG_WRITE(some more registers)
    - SET_DRAW_STATE() ?
    - REG_WRITE(a bunch more regs)
  - fd6_emit_lrz_flush
    - EVENT_WRITE(LRZ_FLUSH)
  - fd6_cache_flush
  - SKIP_IB2_ENABLE_GLOBAL(0)
  - fd_wfi
    - WAIT_FOR_IDLE()
  - REG_WRITE(RB_CCU_CNTL, 0x7c40004)
  - emit_zs
    - REG_WRITE(RB_DEPTH_BUFFER_INFO, ...)
    - REG_WRITE(GRAS_SU_DEPTH_BUFFER_INF, ...)
    - REG_WRITE(GRAS_LRZ_BUFFER_BASE_LO, ...)
    - REG_WRITE(RB_STENCIL_INFO, ...)
  - emit_mrt
    - for each render target
      - REG_WRITE(RB_MRT_BUF_INFO, ...): addr, format, tiling, stride
      - REG_WRITE(SP_FS_MRT_REG): format
    - REG_WRITE(RB_SRGB_CNTL): sRGB rendering
    - REG_WRITE(SP_SRGB_CNTL): sRGB rendering
    - REG_WRITE(SRB_RENDER_COMPONENTS): which MRTs are on
    - REG_WRITE(SP_FS_RENDER_COMPONENTS): which MRTs are on
  - disable_msaa
    - REG_WRITE(SP_TP_RAS_MSAA_CNTL, ...);
    - REG_WRITE(GRAS_RAS_MSAA_CNTL, ...);
    - REG_WRITE(RB_RAS_MSAA_CNTL, ...);
  - set_bin_size(BINNING_PASS)
    - REG_WRITE(GRAS_BIN_CONTROL): bin width/height
    - REG_WRITE(RB_BIN_CONTROL): bin width/height
    - REG_WRITE(RB_BIN_CONTROL2): bin width/height
  - update_render_cntl(true)
    - CP_ONE_REG_WRITE(RB_RENDER_CNTL, ...)
  - emit_binning_pass
    - set_scissor
      - REG_WRITE(GRAS_SC_WINDOW_SCISSOR_TL)
      - REG_WRITE(GGRAS_RESOLVE_CNTL_1)
    - emit_marker6
    - SET_MARKER(RM6_BINNING): switch to binning mode
    - emit_marker6
    - SET_VISIBILITY_OVERRIDE(1)
    - SET_MODE(1)
    - WAIT_FOR_IDLE()
    - REG_WRITE(VFD_MODE_CNTL, BINNING_PASS)
    - update_vsc_pipe
      - REG_WRITE(VSC_BIN_SIZE, ...)
      - REG_WRITE(VSC_BIN_COUNT)
      - REG_WRITE(VSC_PIPE_CONFIG_REG, ...)
      - REG_WRITE(VSC_PIPE_DATA2_ADDRESS_LO, ...)
      - REG_WRITE(VSC_PIPE_DATA_ADDRESS_LO, ...)
    - REG_WRITE(unknown regs)
    - EVENT_WRITE(0x2c)
    - REG_WRITE(RB_WINDOW_OFFSET)
    - REG_WRITE(SP_TP_WINDOW_OFFSET)
    - fd6_emit_ib(batch->draw)
      - emit_marker6
      - INDIRECT_BUFFER(address, size)
      - emit_marker6
    - SET_DRAW_STATE()
    - EVENT_WRITE(0x2d)
    - EVENT_WRITE(CACHE_FLUSH_TS, blit_mem, 0x0)
    - fd_wfi
  - set_bin_size(USE_VIZ)
  - REG_WRITE(VFD_MODE_CNTL, 0)
  - update_render_cntl(false)
- for each tile
  - fd6_emit_tile_prep
    - SET_MARKER(MODE(0x7))
    - emit_marker6
    - SET_MARKER(RM6_GMEM)
    - emit_marker6
    - set_scissor
    - set_window_offset
      - REG_WRITE(RB_WINDOW_OFFSET)
      - REG_WRITE(RB_WINDOW_OFFSET2)
      - REG_WRITE(SP_WINDOW_OFFSET)
      - REG_WRITE(SP_TP_WINDOW_OFFSET)
    - REG_WRITE(VPC_SO_OVERRIDE, DISABLE)
    - WAIT_FOR_ME()
    - SET_VISIBILITY_OVERRIDE(0)
    - SET_MODE(0)
    - SET_BIN_DATA5()
  - fd6_emit_tile_mem2gmem
    - no-op
  - fd6_emit_tile_renderprep
    - fd6_emit_ib(batch->tile_setup)
  - fd6_emit_ib(batch->draw)
  - fd6_emit_tile_gmem2mem
    - fd6_emit_ib(batch->tile_fini)
- fd6_emit_tile_fini
  - REG_WRITE(GRAS_LRZ_CNTL, ENABLE | UNK3)
  - EVENT_WRITE(LRZ_FLUSH)
  - fdm_event_write(CACHE_FLUSH_TS)
    - EVENT_WRITE(CACHE_FLUSH_TS, blit_mem, seqno)
- This is it.  Let's visit IBs
- fd6_clear_lrz creates batch->lrz_clear
  - emit_marker6()
  - SET_MARKER(RM6_BYPASS)
  - emit_marker6()
  - REG_WRITE(RB_CCU_CNTL, 0x10000000)
  - emit_marker6()
  - SET_MARKER(0xc)
  - emit_marker6()
  - REG_WRITE(unknown)
  - REG_WRITE(SP_PS_2D_SRC_INFO)
  - REG_WRITE(a bunch more)
  - REG_WRITE(RB_2D_DST_INFO)
  - REG_WRITE(more)
  - EVENT_WRITE(0x3f)
  - WAIT_FOR_IDLE()
  - REG_WRITE(unknown)
  - BLIT(SCALE)
  - REG_WRITE(unknown)
  - EVENT_WRITE(0x1d)
  - EVENT_WRITE(FACENESS_FLUSH)
  - EVENT_WRITE(CACHE_FLUSH_TS)
  - fd6_cache_flush
- CP_BLIT: sys mem to sys mem 2D blit
- EVENT_WRITE(BLIT): sys mem <-> gmem blit
  - GMEM mode (GMEM is dest): sys -> gmem
  - RESOLVE mode (GMEM is src): gmem -> sys
  - CP_SET_MARKER sets the mode
  - RB_BLIT_xxx to set up sys mem address

## Formats

- mesa formats
  - MESA_FORMAT_<COMPONENTS>_<TYPE>
  - <COMPONENTS> convention
    - packed: in LSB to MSB order
    - array: in low to high order
- pipe formats
  - PIPE_FORMAT_<COMPONENTS>_<TYPE>
  - <COMPONENTS> convention
    - packed (is_bitmask): in LSB to MSB order
    - array (!is_bitmask): in low to high order
- vk formats
  - VK_FORMAT_<COMPONENTS>_<TYPE>
  - <COMPONENTS> convention
    - packed: in MSB to LSB order
    - array: in low to high order
- msm formats
  - little-endian
  - for (MSB A R G B LSB), swap is (W X Y Z)
  - for (MSB A B G R LSB), swap is (W Z Y X)
  - in other words, swap labels each component in msb-to-lsb order
    - X means R (or D)
    - Y means G (or S)
    - Z means B
    - W means A
  - when format has 3 components, W is at the leading position
  - when format has 2 components, WZ is at the leading positions
  - when format has 1 components, WZY is at the leading positions

## turnip

- implicit fencing
  - turnip uses soft pin for all bos (i.e., their iova are known and fixed)
  - turnip keeps tracks of ALL bos allocated from a device
    - `tu_bo_init` adds the bo to `dev->bo_list`
  - all of them are marked `MSM_SUBMIT_BO_READ` and `MSM_SUBMIT_BO_WRITE`
  - when submit, all bos are added to the submit
  - in `submit_fence_sync`, the kernel driver waits for implicit fences
    - `msm_gem_sync_object` waits for existing implicit fences
  - in `submit_pin_objects`, the kernel driver pins all bos
    - `msm_gem_pin_iova` pins the pages of each bo and maps them to iommu
  - in `msm_gpu_submit`, the kernel sets up implcit fencing
    - because `MSM_SUBMIT_BO_WRITE` is always set, it calls
      `dma_resv_add_excl_fence` on all bos
