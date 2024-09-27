ARM Mali
========

## History

- 2007: Utgard
  - GLES 2.0
  - non-unified, vector isa
  - Mali-{200,300,400,450}
- 2010: Midgard
  - GLES 3.1
  - unified, vector isa
  - 1 to 4 unified shader cores
  - 2 or 4 ALUs per shader core
  - 4x or 16x FSAA
  - 1 MMU
  - 32KB to 256KB L2
  - 16x16 tiles
  - Mali-{T604,T658}
- 2012: Midgard 2nd gen
  - ASTC
  - Mali-{T622,T624,T628,T678}
- 2013: Midgard 3nd gen
  - GLES 3.2
  - 4x to 16x MSAA
  - 256KB to 2048MB L2
  - 1 to 16 cores
  - AFBC framebuffer compression
  - Mali-{T720,T760}
- 2014: Midgard 4nd gen
  - Mali-{T820,T830,T860,T880}
- 2016: Bifrost
  - Unified shaders with quad vectorization
  - Scalar ISA
  - Clauses execution
  - 1 to 32 cores
  - Full cache coherency
  - Mali-{G31,G51,G71}
- 2017: Bifrost 2nd gen
  - Mali-{G52,G72}
  - e.g., corsola
- 2018: Bifrost 3nd gen
  - 8 execution lanes per engine (up from 4). Doubled pixel and texel
    throughput.
  - Mali-G76
- 2019: Valhall
  - New superscalar engine
  - Simplified scalar ISA
  - New dynamic scheduling
  - Mali-{G57,G77}
  - e.g., cherry
- 2020: Valhall 2nd gen
  - Asynchronous clock domains
  - New FMA units and increase Tiler throughput
  - Arm Frame Buffer Compression (AFBC) 1.3
  - Mali-{G68,G78}
  - e.g., pixel 6
- 2021: Valhall 3rd gen
  - Larger shader cores (2x compared to Valhall 2nd Gen)
  - New GPU frontend, Command Stream Frontend (CSF) replaces the Job Manager (JM)
  - Mali-{G310,G510,G610,G710}
  - e.g., pixel 7, dimensity 9000
- 2022: Valhall 4th gen
  - Ray Tracing support (hardware-based)
  - Variable Rate Shading
  - New Execution Engine, with doubled the FMA block, Matrix Multiply instruction support, and PPA improvements
  - Arm Fixed Rate Compression (AFRC)
  - Mali-{G615,G715}, Immortalis-G715
  - e.g., pixel 8, pixel 9, dimensity 9200
- 2023: 5th gen
  - Deferred vertex shading (DVS) pipeline
  - Mali-{G620,G720}, Immortalis-G720
  - e.g., dimensity 9300
- 2024: 5th gen
  - Mali-{G625,G725}, Immortalis-G925
  - e.g., dimensity 9400?

## Resources

- <https://developer.arm.com/documentation/102849/latest/>
- <https://developer.arm.com/documentation/101574/latest/>
- Performance Counters
  - <https://developer.arm.com/documentation/102812/latest/>
  - <https://developer.arm.com/documentation/107775/latest/>
  - <https://developer.arm.com/documentation/108081/latest>
  - <https://developer.arm.com/documentation/109780/latest>
  - <https://developer.arm.com/documentation/109793/latest/>
- <https://community.arm.com/arm-community-blogs/tags/GPU>

## Identification

- `GPU_ID` register
  - bit 0..3: status
  - bit 4..11: minor
  - bit 12..15: major
  - bit 16..31: id
    - bit 16..19: product
    - bit 20..23: arch revision
    - bit 24..27: arch minor
    - bit 28..31: arch major
- `pan_arch()`
  - midgard: v4 and v5
    - the 16-bit id is less than `0x1000`
    - a table is used to look up the arch versions
  - bifrost: v6 and v7
    - `id >> 12` is the arch versions
  - valhall: v9 and v10
- kbase uses arch major and product for identification
  - also <https://gitlab.freedesktop.org/panfrost/valhall-docs/-/blob/main/Appendix.md>
  - `GPU_ID_PRODUCT_TMIX` is `GPU_ID_MODEL_MAKE(6, 0)`
    - Mali-G71, bifrost
    - mimir
  - `GPU_ID_PRODUCT_THEX` is `GPU_ID_MODEL_MAKE(6, 1)`
    - Mali-G72, bifrost 2nd
    - heidmall
  - `GPU_ID_PRODUCT_TSIX` is `GPU_ID_MODEL_MAKE(7, 0)`
    - Mali-G51, bifrost
    - sigurd
  - `GPU_ID_PRODUCT_TDVX` is `GPU_ID_MODEL_MAKE(7, 3)`
    - Mali-G31, bifrost
    - dvalin
  - `GPU_ID_PRODUCT_TNOX` is `GPU_ID_MODEL_MAKE(7, 1)`
    - Mali-G76, bifrost 3rd
    - norr
  - `GPU_ID_PRODUCT_TGOX` is `GPU_ID_MODEL_MAKE(7, 2)`
    - Mali-G52, bifrost 2nd
    - gondul
  - `GPU_ID_PRODUCT_TTRX` is `GPU_ID_MODEL_MAKE(9, 0)`
    - Mali-G77, valhall
    - trym
  - `GPU_ID_PRODUCT_TNAX` is `GPU_ID_MODEL_MAKE(9, 1)`
    - Mali-G57, valhall
    - natt-a
  - `GPU_ID_PRODUCT_TBEX` is `GPU_ID_MODEL_MAKE(9, 2)`
    - Mali-G78, valhall 2nd
    - borr
  - `GPU_ID_PRODUCT_LBEX` is `GPU_ID_MODEL_MAKE(9, 4)`
    - Mali-G68, valhall 2nd
    - ottr
  - `GPU_ID_PRODUCT_TBAX` is `GPU_ID_MODEL_MAKE(9, 5)`
    - Mali-G78AE
    - borr-ae
  - `GPU_ID_PRODUCT_TODX` is `GPU_ID_MODEL_MAKE(10, 2)`
    - Mali-G710, valhall 3rd gen
    - odin?
  - `GPU_ID_PRODUCT_TGRX` is `GPU_ID_MODEL_MAKE(10, 3)`
    - Mali-G510, valhall 3rd gen
  - `GPU_ID_PRODUCT_TVAX` is `GPU_ID_MODEL_MAKE(10, 4)`
    - Mali-G310, valhall 3rd gen
  - `GPU_ID_PRODUCT_LODX` is `GPU_ID_MODEL_MAKE(10, 7)`
    - Mali-G610, valhall 3rd gen
  - `GPU_ID_PRODUCT_TTUX` is `GPU_ID_MODEL_MAKE(11, 2)`
    - Mali-G715, valhall 4th gen
      - Immortalis if high core count and raytracing
      - G615 if low core count
    - turse?
  - `GPU_ID_PRODUCT_LTUX` is `GPU_ID_MODEL_MAKE(11, 3)`
    - Mali-G615, valhall 4th gen
  - `GPU_ID_PRODUCT_TTIX` is `GPU_ID_MODEL_MAKE(12, 0)`
    - Mali-G720, 5th gen
      - Immortalis if high core count and raytracing
      - G620 if low core count
    - titan?
  - `GPU_ID_PRODUCT_LTIX` is `GPU_ID_MODEL_MAKE(12, 1)`
    - Mali-G620, 5th gen
  - `GPU_ID_PRODUCT_TKRX` is `GPU_ID_MODEL_MAKE(13, 0)`
    - Mali-G725
      - Immortalis-G925 if high core count and raytracing
      - G625 if low core count
  - `GPU_ID_PRODUCT_LKRX` is `GPU_ID_MODEL_MAKE(13, 1)`
    - Mali-G625

## Utgard

- GLES 2.0
- non-unified shader cores
- one vertex core
  - the vertex core and the tiler share a small L2
  - shade one or two vertices serially
  - output to fixed-function tiler for primitive assembly, clipping, cullling,
    and tile list generation
- multiple fragment cores
  - they share a larger L2
  - L2 size is usually `num_cores * 32KB`
  - data flow within a core
    - tile list reader
    - rasterizer
    - early zs tester
    - fragment thread creater
    - threads doing load, texture, arithmetic, and then retire
    - late zs tester
    - blender
    - tile memory
    - tile writeback
  - a core can run up to 128 threads to hide cache misses and memory latency
  - ISA is SIMD and operate on vec4 fp16

## Midgard

- GLES 3.2
- unified shader cores
- a L2 cache shared by all shader cores
  - size is `num_cores * (32KB or 64KB)`
- given a render pass
  - the driver submits geometry workload to HW first
  - the geometry queue block dispatches the workload to shader cores
  - the driver then submits fragment workload to HW
  - the fragment queue block dispatches the workload to shader cores
- a shader core consists of
  - fixed-function part
    - tile list reader
    - rasterizer
    - early zs tester
    - vertex or fragment thread creator
  - programmable tripipe
    - thread pool
    - three types of pipelines
      - one load/store pipeline w/ L1
      - one texture pipeline w/ L1
      - one or more arithmetic pipelines
	- 128-bit SIMD (4 fp32 or 8 fp16 or 16 i8)
    - thread retire
  - more fixed-function parts
    - late zs tester
    - blender
    - tile memory
    - tile writeback
- 16x16 tiles
- 4KB on-chip memory

## Bifrost

- "The Bifrost GPU architecture and the ARM Mali-G71 GPU"
  - by Jem Davies
- a L2 cache shared by all shader cores
  - size is `num_cores * (64KB or 128KB)`
  - full coherency
- up to 32 unified shader cores, where each core consists of
  - a compute frontend
    - quad creator
  - a fragment frontend
    - tile list reader
    - rasterizer
    - early zs tester
    - quad creator
  - a quad manager
    - warp manager
  - up to three execution engines, each
    - 32-bit scalar ISA
    - bifrost uses 4-lane quad-parallel vectorization (SIMT...)
      - a quad is 4 scalar threads executed in lockdep
      - one quad at a time executes in each pipeline stage
      - each thread fills a 32-bit lane of the hardware
      - 4 threads all doing a vec fp32 add take 3 cycles
    - midgard uses 4-lane SIMD vectorization
      - one thread at a time executes in each pipeline stage
      - each thread must fill the width of the hardware
      - 4 threads all doing a SIMD vec3 fp32 add take 4 cycles
  - load/store unit
  - attribute unit
  - varying unit
  - texture units (one or two)
  - zs/blend units (one or two)
    - late zs tester
    - blender
    - tile memory
    - tile writeback
- geometry pipeline
  - instead of 'shade -> assembly -> clip/cull`
  - it does 'assembly -> position shade -> clip/cull -> varying shade`
  - specifically,
    - tiler fetches indices for primitive assembly
    - position shading fetches and transforms only positions
    - tiler clips/culls and stores
      - transformed positions
      - polyton list 
    - varying shading fetches attributes and stores varyings
    - fraghment shading reads back
      - transformed positions
      - polyton list 
      - varyings
  - it is beneficial to have two packed buffers, one for positions and one for
    attributes

## Valhall

- G77 high-level features
  - better AR and ML
  - a new superscalar engine
  - a simplified scalar ISA that is more compiler-friendly
  - dynamic scheduling of instructions
  - data structures that are more Vulkan-friendly
  - 16-wide warps, 32 lanes
    - G76 has 8-wide warps, 24 lanes
  - quad texture mapper, 4 texels/cycle
    - G76 is 2 texels/cycle
  - AFBC 1.3
- a shader core consists of
  - a compute frontend
  - a fragment frontend
  - a manager
  - an execution engine
    - front-end
      - create/retire warps
      - track states for warps
    - scheduler
      - issue instructions
    - two processsing units, each
      - 16-wide FMA unit
      - 16-wide CVT unit (convert unit)
      - 4-wide special function unit
  - an attribute unit
  - a varying unit
  - 4x texture unit
  - a load-store cache
    - replaces the load-store unit

## kbase

- <https://developer.arm.com/downloads/-/mali-drivers/5th-gen-gpu-architecture-kernel>
  - there are separated downloads for
    - Bifrost 3rd Gen GPU Architecture Kernel
    - Valhall 4th Gen GPU Architecture Kernel
    - 5th Gen GPU Architecture Kernel
  - but they are all the same code
- `Kconfig`
  - `MALI_MIDGARD=y`
  - `MALI_PLATFORM_NAME="devicetree"`
  - `MALI_REAL_HW=y`
  - `MALI_CSF_SUPPORT=y`
  - `MALI_DEVFREQ=y`
  - `LARGE_PAGE_SUPPORT=y`
  - `PAGE_MIGRATION_SUPPORT=y`
- `Kbuild`
  - `Makefile` is preferred, but if `Kbuild` exists, it is used
  - the top-level `Kbuild` includes the subdir `Kbuild`s
  - `CONFIG_MALI_CSF_SUPPORT` causes different files to be compiled
- comparing to
  <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/refs/heads/chromeos-6.6/drivers/gpu/arm/>
  - the subdir is `drivers/gpu/midgard` rather than `drivers/gpu/mali`
  - the main config is `CONFIG_MALI_MIDGARD` rather than `CONFIG_MALI`
  - no mtk-specific or cros-specific code

## kbase init

- `kbase_driver_init` registers `kbase_platform_driver`
  - driver name is `mali`
  - id table has `arm,mali-valhall`, `arm,mali-bifrost`, etc.
- `kbase_device_init` initializes the device
  - `kbase_get_irqs`
    - `platform_get_irq_byname` gets 3 IRQs: `JOB`, `MMU`, and `GPU`
  - `registers_map`
    - `platform_get_resource` gets `IORESOURCE_MEM`
    - `kbase_common_reg_map` requests and ioremaps the region
  - `power_control_init` gets the regulators and clks
  - `kbase_device_io_history_init` inits `kbdev->io_history`
  - `kbase_device_early_init`
    - `kbasep_platform_device_init` depends on `MALI_PLATFORM_NAME`
    - `kbase_pm_runtime_init` depends on `MALI_PLATFORM_NAME`
    - `kbase_gpuprops_parse_gpu_id` reads and parses `GPU_ID` reg
    - `kbase_regmap_init` inits `kbdev->regmap`
    - `kbase_hw_set_features_mask` inits `kbdev->hw_features_mask` based on
      `GPU_ID`
    - `kbase_gpuprops_init` reads various regs to init `kbdev->gpu_props`
      - `SHADER_PRESENT`, `TILER_PRESENT`, `L2_PRESENT`, etc.
    - `kbase_hw_set_issues_mask` inits hw workarounds
    - `kbase_install_interrupts` registers irq handlers
      - `kbase_job_irq_handler`
      - `kbase_mmu_irq_handler`
      - `kbase_gpu_irq_handler`
  - `kbase_backend_time_init` inits `kbdev->backend_time`
  - `kbase_device_misc_init` inits various stuff
  - `kbase_device_pcm_dev_init`
  - `kbase_ctx_sched_init`
  - `kbase_mem_init`
    - `kbdev->mgm_dev` is set to `kbase_native_mgm_dev`
    - if there is a `arm,physical-memory-group-manager` node and if
      `memory_group_manager_driver` is enabled, `kbdev->mgm_dev` is set to the
      driver instead
  - `kbase_csf_protected_memory_init`
  - `kbase_device_coherency_init` inits `kbdev->system_coherency`
    - usually to `COHERENCY_NONE`
  - `kbase_protected_mode_init`
  - `kbase_device_list_init`
  - `kbase_device_timeline_init` inits `kbdev->timeline`
  - `kbase_clk_rate_trace_manager_init`
  - `kbase_device_hwcnt_watchdog_if_init`
  - `kbase_device_hwcnt_backend_csf_if_init`
  - `kbase_device_hwcnt_backend_csf_init`
  - `kbase_device_hwcnt_context_init`
  - `kbase_csf_early_init` inits `kbdev->csf`
  - `kbase_backend_late_init`
    - `kbase_backend_devfreq_init` inits devfreq when `CONFIG_MALI_DEVFREQ` is
      enabled
      - `CONFIG_MALI_DEVFREQ` replaces `CONFIG_MALI_MIDGARD_DVFS`
      - `kbase_devfreq_init` calls `kbase_ipa_init`
        - IPA stands for Intelligent Power Allocation
  - `kbase_csf_late_init`
  - `kbase_debug_csf_fault_init`
  - `kbase_device_debugfs_init`
  - `kbase_csf_fence_timer_debugfs_init`
  - `kbase_sysfs_init` inits `kbdev->mdev` and sysfs
  - `kbase_device_misc_register` registers `kbdev->mdev`
  - `kbase_gpuprops_populate_user_buffer` inits gpu props to be queried by
    userspace
  - `kbase_device_late_init`
- when `/dev/mali0` is opened, `kbase_open` is called
  - `kbase_device_firmware_init_once` loads `mali_csffw.bin`
  - `kbase_file_new` creates a `kbase_file`

## kbase ioctls

- ioctls are handled by `kbase_ioctl`
- `KBASE_IOCTL_VERSION_CHECK` is handled by `kbase_api_handshake`
  - it negotiates uapi version
- `KBASE_IOCTL_SET_FLAGS` is handled by `kbase_api_set_flags`
  - `kbase_file_create_kctx` creates the ctx
- `KBASE_IOCTL_KINSTR_PRFCNT_ENUM_INFO` is handled by `kbase_api_kinstr_prfcnt_enum_info`
- `KBASE_IOCTL_KINSTR_PRFCNT_SETUP` is handled by `kbase_api_kinstr_prfcnt_setup`
- `KBASE_IOCTL_GET_GPUPROPS` is handled by `kbase_api_get_gpuprops`
  - queries all gpu props
- `KBASE_IOCTL_MEM_ALLOC` is handled by `kbase_api_mem_alloc`
  - it calls `kbase_api_mem_alloc_ex`
  - it returns `gpu_va` to the userspace, which is also used as the handle to
    the bo
  - `kbase_region_tracker_find_region_base_address` maps `gpu_va` to
    `kbase_va_region`
- `KBASE_IOCTL_MEM_QUERY` is handled by `kbase_api_mem_query`
- `KBASE_IOCTL_MEM_FREE` is handled by `kbase_api_mem_free`
- `KBASE_IOCTL_DISJOINT_QUERY` is handled by `kbase_api_disjoint_query`
- `KBASE_IOCTL_GET_DDK_VERSION` is handled by `kbase_api_get_ddk_version`
  - it returns the DDK version string, such as `K:r51p0-00eac0(GPL)`
- `KBASE_IOCTL_MEM_JIT_INIT` is handled by `kbase_api_mem_jit_init`
- `KBASE_IOCTL_MEM_EXEC_INIT` is handled by `kbase_api_mem_exec_init`
- `KBASE_IOCTL_MEM_SYNC` is handled by `kbase_api_mem_sync`
- `KBASE_IOCTL_MEM_FIND_CPU_OFFSET` is handled by `kbase_api_mem_find_cpu_offset`
- `KBASE_IOCTL_MEM_FIND_GPU_START_AND_OFFSET` is handled by `kbase_api_mem_find_gpu_start_and_offset`
- `KBASE_IOCTL_GET_CONTEXT_ID` is handled by `kbase_api_get_context_id`
- `KBASE_IOCTL_TLSTREAM_ACQUIRE` is handled by `kbase_api_tlstream_acquire`
- `KBASE_IOCTL_TLSTREAM_FLUSH` is handled by `kbase_api_tlstream_flush`
- `KBASE_IOCTL_MEM_COMMIT` is handled by `kbase_api_mem_commit`
- `KBASE_IOCTL_MEM_ALIAS` is handled by `kbase_api_mem_alias`
- `KBASE_IOCTL_MEM_IMPORT` is handled by `kbase_api_mem_import`
- `KBASE_IOCTL_MEM_FLAGS_CHANGE` is handled by `kbase_api_mem_flags_change`
- `KBASE_IOCTL_MEM_PROFILE_ADD` is handled by `kbase_api_mem_profile_add`
- `KBASE_IOCTL_STICKY_RESOURCE_MAP` is handled by `kbase_api_sticky_resource_map`
- `KBASE_IOCTL_STICKY_RESOURCE_UNMAP` is handled by `kbase_api_sticky_resource_unmap`
- `KBASE_IOCTL_GET_CPU_GPU_TIMEINFO` is handled by `kbase_api_get_cpu_gpu_timeinfo`
  - returns `CYCLE_COUNT`, `TIMESTAMP`, and `ktime_get_raw_ts64`
- `KBASE_IOCTL_CONTEXT_PRIORITY_CHECK` is handled by `kbasep_ioctl_context_priority_check`
  - it updates ctx priority
- `KBASE_IOCTL_SET_LIMITED_CORE_COUNT` is handled by `kbase_ioctl_set_limited_core_count`
  - it updates `kctx->limited_core_mask`
- `MALI_USE_CSF`
  - `KBASE_IOCTL_MEM_ALLOC_EX` is handled by `kbase_api_mem_alloc_ex`
  - `KBASE_IOCTL_CS_EVENT_SIGNAL` is handled by `kbasep_cs_event_signal`
  - `KBASE_IOCTL_CS_QUEUE_REGISTER` is handled by `kbasep_cs_queue_register`
    - registers a mem as a queue and creates `kbase_queue` for it
  - `KBASE_IOCTL_CS_QUEUE_REGISTER_EX` is handled by `kbasep_cs_queue_register_ex`
  - `KBASE_IOCTL_CS_QUEUE_TERMINATE` is handled by `kbasep_cs_queue_terminate`
  - `KBASE_IOCTL_CS_QUEUE_BIND` is handled by `kbasep_cs_queue_bind`
    - adds a queue to a queue group
  - `KBASE_IOCTL_CS_QUEUE_KICK` is handled by `kbasep_cs_queue_kick`
  - `KBASE_IOCTL_CS_QUEUE_GROUP_CREATE` is handled by `kbasep_cs_queue_group_create`
    - creates a `kbase_queue_group`
  - `KBASE_IOCTL_CS_QUEUE_GROUP_TERMINATE` is handled by `kbasep_cs_queue_group_terminate`
  - `KBASE_IOCTL_KCPU_QUEUE_CREATE` is handled by `kbasep_kcpu_queue_new`
    - creates a `kbase_kcpu_command_queue`
  - `KBASE_IOCTL_KCPU_QUEUE_DELETE` is handled by `kbasep_kcpu_queue_delete`
  - `KBASE_IOCTL_KCPU_QUEUE_ENQUEUE` is handled by `kbasep_kcpu_queue_enqueue`
  - `KBASE_IOCTL_QUEUE_GROUP_CLEAR_FAULTS` is handled by `kbasep_queue_group_clear_faults`
  - `KBASE_IOCTL_CS_TILER_HEAP_INIT` is handled by `kbasep_cs_tiler_heap_init`
  - `KBASE_IOCTL_CS_TILER_HEAP_TERM` is handled by `kbasep_cs_tiler_heap_term`
  - `KBASE_IOCTL_CS_GET_GLB_IFACE` is handled by `kbase_ioctl_cs_get_glb_iface`
  - `KBASE_IOCTL_CS_CPU_QUEUE_DUMP` is handled by `kbasep_ioctl_cs_cpu_queue_dump`
  - `KBASE_IOCTL_READ_USER_PAGE` is handled by `kbase_ioctl_read_user_page`
- `!MALI_USE_CSF`
  - `KBASE_IOCTL_JOB_SUBMIT` is handled by `kbase_api_job_submit`
  - `KBASE_IOCTL_POST_TERM` is handled by `kbase_api_post_term`
  - `KBASE_IOCTL_SOFT_EVENT_UPDATE` is handled by `kbase_api_soft_event_update`
  - `KBASE_IOCTL_KINSTR_JM_FD` is handled by `kbase_api_kinstr_jm_fd`
- `CONFIG_SYNC_FILE`
  - `KBASE_IOCTL_STREAM_CREATE` is handled by `kbase_api_stream_create`
    - returns an fd
  - `KBASE_IOCTL_FENCE_VALIDATE` is handled by `kbase_api_fence_validate`
- `CONFIG_MALI_CINSTR_GWT`
  - `KBASE_IOCTL_CINSTR_GWT_START` is handled by `kbase_gpu_gwt_start`
  - `KBASE_IOCTL_CINSTR_GWT_STOP` is handled by `kbase_gpu_gwt_stop`
  - `KBASE_IOCTL_CINSTR_GWT_DUMP` is handled by `kbase_gpu_gwt_dump`

## kbase pm

- `kbase_pm_ops` is the `dev_pm_ops`
- `kbase_pm_callback_conf` defines platform-specific callbacks
- `kbase_backend_late_init` is called on init
  - `kbase_hwaccess_pm_powerup`
    - `kbase_pm_init_hw`
      - `callback_power_on`
    - `callback_power_runtime_gpu_active`
    - `kbase_pm_do_poweron`
      - `kbase_pm_clock_on`
        - `callback_power_on`
- `kbase_device_suspend` is called on system suspend
  - `kbase_pm_suspend`
    - `kbase_csf_scheduler_pm_suspend` suspends CSF
    - `kbase_hwaccess_pm_suspend`
      - `kbase_pm_do_poweroff_sync`
        - `kbase_pm_update_state`
        - `callback_power_runtime_gpu_idle`
        - `kbase_pm_clock_off`
          - `callback_power_off`
      - `callback_power_suspend`
  - `kbase_devfreq_enqueue_work` calls `devfreq_suspend_device`
- `kbase_device_resume` is called on system resume
  - `kbase_pm_resume`
    - `kbase_hwaccess_pm_resume`
      - `kbase_pm_do_poweron`
        - `kbase_pm_clock_on`
          - `callback_power_resume` or `callback_power_on`
          - `kbase_pm_update_state`
          - `callback_power_runtime_gpu_active`
    - `kbase_csf_scheduler_pm_resume`
  - `kbase_devfreq_enqueue_work` calls `devfreq_resume_device`
- `kbase_device_runtime_idle` is called to see if a device can be runtime
  suspended
  - `callback_power_runtime_idle` if provided
  - otherwise, `pm_runtime_mark_last_busy` and returns 0
- `kbase_device_runtime_suspend` is called for runtime suspend
  - `kbase_pm_handle_runtime_suspend`
    - `kbase_pm_clock_off`
  - `kbase_devfreq_enqueue_work` calls `devfreq_suspend_device`
  - `callback_power_runtime_off`
- `kbase_device_runtime_resume` is called for runtime resume
  - `callback_power_runtime_on`
  - `kbase_devfreq_enqueue_work` calls `devfreq_resume_device`
- `mali_kbase_pm.h`
  - `kbase_pm_context_active` increments `kbdev->pm.active_count`
    - this powers up gpu blocks on first use
  - `kbase_pm_context_idle` decrements `kbdev->pm.active_count`
    - this powers down gpu blocks on last use
  - `kbase_pm_suspend` calls `kbase_pm_driver_suspend`
    - this is called in response to system suspend
  - `kbase_pm_resume` calls `kbase_pm_driver_resume`
    - this is called in response to system resume
- `mali_kbase_hwaccess_pm.h`
  - `kbase_hwaccess_pm_init`
    - `kbase_pm_ca_init` core availability
    - `kbase_pm_policy_init` defaults to `kbase_pm_coarse_demand_policy_ops`,
      which uses `kbase_pm_is_active` to check `kbdev->pm.active_count`
    - `kbase_pm_state_machine_init` inits the pm state machine for power
      transitions
    - `kbdev->pm.backend.gpu_sleep_allowed`
      - v11+ supports `KBASE_HW_FEATURE_GPU_SLEEP` and enables
        `KBASE_GPU_SUPPORTS_GPU_SLEEP`
        - but early v11 has `KBASE_HW_ISSUE_TURSEHW_1997`
        - on gpu idle,
          - before v11, it powers down mcu and l2
          - after v11, it pauses mcu and powers down l2?
      - v12+ (and late v11) supports `KBASE_GPU_SUPPORTS_FW_SLEEP_ON_IDLE`
        - on gpu idle, mcu pauses itself automatically?
  - `kbase_hwaccess_pm_term` cleans up
  - `kbase_hwaccess_pm_powerup` powers up gpu, inits, and powers it down
  - `kbase_hwaccess_pm_halt`
  - `kbase_hwaccess_pm_suspend` is caleld from `kbase_pm_suspend`
  - `kbase_hwaccess_pm_resume` is caleld from `kbase_pm_resume`
  - `kbase_hwaccess_pm_gpu_active` performs all actions to get gpu active
    - it is called on first `kbase_pm_context_active`
    - `kbase_pm_do_poweron`
  - `kbase_hwaccess_pm_gpu_idle` performs all actions to get gpu idle
    - it is called on last `kbase_pm_context_idle`
    - `kbase_pm_do_poweroff`
- `backend/gpu/mali_kbase_pm_internal.h`
  - `kbase_pm_get_present_cores` returns `{L2,SHADER,TILER,STACK}_PRESENT`
  - `kbase_pm_get_active_cores` returns `{L2,SHADER,TILER,STACK}_PWRACTIVE`
  - `kbase_pm_get_trans_cores` returns `{L2,SHADER,TILER,STACK}_PWRTRANS`
  - `kbase_pm_get_ready_cores` returns `{L2,SHADER,TILER,STACK}_READY`
- `kbase_pm_l2_update_state` handles L2 power state changes
  - when `kbase_pm_is_l2_desired` is true, it transitions from `KBASE_L2_OFF`
    to `KBASE_L2_ON`
    - if `KBASE_L2_OFF`, writes `L2_CONFIG` and `L2_PWRON`
    - if `KBASE_L2_PEND_ON`, checks `L2_PWRTRANS` and `L2_READY`
    - if `KBASE_L2_ON_HWCNT_ENABLE`, nothing
    - if `KBASE_L2_ON`, done
  - when `kbase_pm_is_l2_desired` is false, it transitions from `KBASE_L2_ON`
    to `KBASE_L2_OFF`
    - if `KBASE_L2_ON`, nothing
    - if `KBASE_L2_ON_HWCNT_DISABLE`, nothing
    - if `KBASE_L2_POWER_DOWN`, writes `L2_PWROFF`
    - if `KBASE_L2_PEND_OFF`, checks `L2_PWRTRANS` and `L2_READY`
    - if `KBASE_L2_OFF`, nothing
- `kbase_pm_mcu_update_state` handles MCU/CSF power state changes
  - it is nop until `kbase_device_firmware_init_once` transitions from
    `KBASE_MCU_OFF` to `KBASE_MCU_ON`
  - when `kbase_pm_is_mcu_desired` returns true, it transitions from
    `KBASE_MCU_OFF` to `KBASE_MCU_ON`
    - if `KBASE_MCU_OFF`, `kbase_csf_firmware_trigger_reload` reboots the fw
    - if `KBASE_MCU_PEND_ON_RELOAD`, it checks for boot complete and
      `kbase_csf_firmware_global_reinit`
    - if `KBASE_MCU_ON_GLB_REINIT_PEND`, it checks for glb init complete and 
    - if `KBASE_MCU_ON_HWCNT_ENABLE`, nothing
    - if `KBASE_MCU_ON`, done
  - when `kbase_pm_is_mcu_desired` returns false, it transitions from
    `KBASE_MCU_ON` to `KBASE_MCU_OFF`
    - if `KBASE_MCU_ON`, nothing
    - if `KBASE_MCU_ON_HWCNT_DISABLE`, nothing
    - on v10 and early v11,
      - if `KBASE_MCU_ON_HALT`, triggers mcu halt
      - if `KBASE_MCU_ON_PEND_HALT`, checks halt complete
      - if `KBASE_MCU_POWER_DOWN`, triggers mcu shutdown
      - if `KBASE_MCU_PEND_OFF`, waits mcu shutdown
      - if `KBASE_MCU_OFF`, done
    - on later v11+,
      - if `KBASE_MCU_ON_SLEEP_INITIATE`, triggers mcu sleep
      - if `KBASE_MCU_ON_PEND_SLEEP`, checks mcu sleep complete
      - if `KBASE_MCU_IN_SLEEP`, done

## kbase csf

- `kbase_device_firmware_init_once` boots MCU on first use
  - `kbase_pm_context_active` requests power on
  - `kbase_csf_firmware_deferred_init` boots mcu
    - `kbase_csf_firmware_load_init` does the booting
      - `kbase_pm_wait_for_l2_powered` waits for l2 powered on
      - write `MCU_CNTRL_AUTO` to `MCU_CONTROL`
      - wait for `JOB_IRQ_GLOBAL_IF` interrupt
    - `kbdev->pm.backend.mcu_state = KBASE_MCU_ON;`
    - `kbdev->csf.firmware_inited = true;`
  - `kbase_pm_context_idle` requests power off

## kbase bo

- interesting `base_mem_alloc_flags`
  - `BASE_MEM_PROT_CPU_RD`
  - `BASE_MEM_PROT_CPU_WR`
  - `BASE_MEM_PROT_GPU_RD`
  - `BASE_MEM_PROT_GPU_WR`
  - `BASE_MEM_PROT_GPU_EX`, executable, for shader binary
  - `BASE_MEM_SAME_VA`, same cpu/gpu va
  - `BASE_MEM_FIXED`, umd-specified fixed va
  - `BASE_MEM_FIXABLE`
- `kbase_mem_alloc`
  - `zone`
    - `SAME_VA_ZONE`, if `BASE_MEM_SAME_VA`
    - `EXEC_FIXED_VA_ZONE`, if fixed and executabe
    - `FIXED_VA_ZONE`, if fixed
    - `EXEC_VA_ZONE`, if executable
    - `CUSTOM_VA_ZONE`, otherwise
- `kbase_region_tracker_init` initializes the zones
  - 0x00000000'00001000..0x00007fff'ffffffff, `SAME_VA_ZONE`
    - `kbase_reg_zone_same_va_init` uses pfn 1 to
      `KBASE_REG_ZONE_EXEC_VA_BASE_64` (47 bits)
  - 0x00008000'00000000..0x00008000'ffffffff, `EXEC_VA_ZONE`
    - `kbase_reg_zone_exec_va_init` uses `KBASE_REG_ZONE_EXEC_VA_SIZE` (4GB)
      following `SAME_VA_ZONE`
  - 0x00008001'00000000..0x00008001'ffffffff, `EXEC_FIXED_VA_ZONE`
    - `kbase_reg_zone_exec_fixed_va_init` uses
      `KBASE_REG_ZONE_EXEC_FIXED_VA_SIZE` (4GB) following `EXEC_VA_ZONE`
  - 0x00008002'00000000..0x0000ffff'ffffffff, `FIXED_VA_ZONE`
    - `kbase_reg_zone_fixed_va_init` uses `KBASE_REG_ZONE_FIXED_VA_END_64`
      (48 bits) following `EXEC_FIXED_VA_ZONE`
  - 0x00000001'00000000..0x000007ff'ffffffff, `CUSTOM_VA_ZONE` (32-bit umd only)
    - `kbase_reg_zone_custom_va_init` uses `KBASE_REG_ZONE_CUSTOM_VA_BASE`
      (4GB) to `KBASE_REG_ZONE_CUSTOM_VA_SIZE` (43 bits)

## kbase v10 and v13

- search for `GPU_ID.*MAKE` for v10 and v13 differences
  - `kbase_regmap_v10_10_init`
    - `CORE_FEATURES`, named `GPU_CORE_FEATURES` in panthor
  - `kbase_regmap_v11_init`
    - `GPU_FEATURES`, 0x60
    - `PRFCNT_FEATURES`, 0x68
    - `DOORBELL_FEATURES`, 0xc0
    - `ASN_HASH_[0-2]`, 0x2c0...
    - `SYSC_PBHA_OVERRIDE[0-3]`, 0x320...
    - `SYSC_ALLOC[0-7]`, 0x340...
    - `LATEST_FLUSH`, named `CSF_GPU_LATEST_FLUSH_ID` in panthor
  - `kbase_regmap_v12_init`
    - `GPU_COMMAND_ARG[01]`, 0xd0...
      - affects `kbase_gpu_cache_flush_pa_range_and_busy_wait`
    - `SHADER_PWRFEATURES`, 0x188
      - affects `kbasep_enable_rtu`
    - `AMBA_FEATURES` replaces `COHERENCY_FEATURES`, named
      `GPU_COHERENCY_FEATURES` in panthor
    - `AMBA_ENABLE` replaces `COHERENCY_ENABLE`, named `GPU_COHERENCY_PROTOCOL`
      in panthor
    - `MCU_FEATURES`, 0x708
  - `kbase_hw_get_issues_for_new_id` reports different issues
    - `KBASE_HW_ISSUE_TURSEHW_2716` and `KBASE_HW_ISSUE_TITANHW_2922`
  - `kbase_hw_set_features_mask` reports different features
    - `KBASE_HW_FEATURE_L2_SLICE_HASH`
    - `KBASE_HW_FEATURE_GPU_SLEEP,`
    - `KBASE_HW_FEATURE_CORE_FEATURES`
    - `KBASE_HW_FEATURE_PBHA_HWU`
    - `KBASE_HW_FEATURE_LARGE_PAGE_ALLOC`
  - `kbase_cache_set_coherency_mode` sets `AMBA_ENABLE` instead of
    `COHERENCY_ENABLE`, which are the same reg anyway
  - `kbase_amba_set_shareable_cache_support` sets `AMBA_ENABLE` when there is
    system coherency
  - `kbasep_pbha_supported` returns true
  - `kbase_get_lock_region_min_size_log2` returns different sizes
  - `mmu_has_flush_skip_pgd_levels` returns true
- search for `arch_major` for v10 and v13 differences

## kbase mediatek platform

- history
  - most mediatek platform updates happened on 5.15 around 2022.02 to 2022.05
    - `git log --oneline -- drivers/gpu/arm/bifrost`
    - `BUG=b:227544156`
  - 4.19
    - added r13p0 to `drivers/gpu/arm/midgard` in 2018.10
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/b303c6455c178a647340cd5e75fddbdf57cc697a>
    - added `drivers/gpu/arm/midgard/platform/mediatek` in 2018.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/d12dc1a835648e06925c9d8f4b4e14017104fe76>
    - added r25p0 to `drivers/gpu/arm/bifrost` in 2020.09
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/faaf0e9b9da24817033645ab22203a8aaa19a678>
    - added `drivers/gpu/arm/bifrost/platform/mediatek` in 2020.09
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/240087743de495f3d99428c9be08a9697089d6d7>
    - switched from midgard to bifrost in 2020.09
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/a3671ea484f1df3d7f128a50b6bc5020c19f3a2d>
  - 5.4
    - added r24p0 to `drivers/gpu/arm/valhall` in 2020.04
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/2017d46f1e629c7c90f3e12f89694e050d9d65fb>
    - added `drivers/gpu/arm/valhall/platform/mt8192` in 2020.09
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/7e8c2ce155f42d1c78dd377f2cc397824965c3a4>
    - updated to r32p0 in 2022.03
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/8db20e11c665a979cbb6e280e334b68f97b706ba>
      - `platform/mt8192` was updated and renamed to `platform/mediatek`
    - updated `drivers/gpu/arm/valhall/platform/mediatek` in 2022.07
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/a639bca8650970eb6f81e886d0e4379b6e23dcda>
    - renamed to `drivers/gpu/arm/mali` in 2022.08
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/2542013a4879415917bc6db20170147df8f4ccba>
  - 5.10
    - added r24p0 to `drivers/gpu/arm/valhall` in 2020.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/f38632197c49e6558fbbd293bcaa11c1cd91c2ca>
    - added `drivers/gpu/arm/valhall/platform/mt8192` in 2020.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/7439a8d1dd860f7778f77a1c17ecd5e9b98fdcc0>
    - added `drivers/gpu/arm/valhall/platform/mt8183` in 2021.01
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/aa4bd4d99fc3300c4c232f71ac2763381abd87bd>
    - merged mt8192/mt8183 to `drivers/gpu/arm/valhall/platform/mediatek` in 2021.05
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/9b36220bb88b97188438aad78b742d0633052ecd>
    - added mt8195 support to `drivers/gpu/arm/valhall/platform/mediatek` in 2021.05
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/d17d194977fd0c139ff9e2cb03a46486ef69963f>
    - added `drivers/gpu/arm/midgard` from 4.19 in 2021.08
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/091c39ecb4a279d5ecdcfc2442d2a9306fcb0344>
      - unused
    - reset to r32p0 in 2022.01
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/79b0a1f26446bb3026f29a6cc0cf40d82112ca85>
    - restored `drivers/gpu/arm/valhall/platform/mediatek` in 2022.01
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/39ff6704c1a9a4da12db11ac202fce301250da7e>
    - renamed to `drivers/gpu/arm/mali` in 2022.08
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/62c8f0ef54f519e80ccb82f9698073947c04db55>
  - 5.15
    - added and removed `drivers/gpu/arm/midgard` in 2021.11 and 2021.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/c1123f9e61e790c85e2e2c53803dc6c21963e1ac>
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/1a7dc1501059f47ca0a69bdc10c49b66abb73edd>
    - added and removed `drivers/gpu/arm/valhall` in 2021.11 and 2021.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/505973e2cbd7c0534bebe61b5472009a8632af92>
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/70a11702ae2c93c57aec31cd3f26ac7856d290b6>
    - added r32p0 to `drivers/gpu/arm/bifrost` in 2021.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/97b1271a1316e9353e0192e8883921e859e08446>
    - added `drivers/gpu/arm/bifrost/platform/mediatek` in 2022.02
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/65b6c72cacf6a558097b70256f3914cf14f04d6b>
    - various updates to `drivers/gpu/arm/bifrost/platform/mediatek` from 2022.02 to 2022.05
    - renamed to `drivers/gpu/arm/mali` in 2022.08
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/0b8e9d9dd88a3b246025c2174151169f4c00b4b6>
  - 6.1
    - added r40p0 to `drivers/gpu/arm/mali` and added
      `drivers/gpu/arm/mali/platform/mediatek` in 2023.02
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/4e899792da6ad557ece992a9fa910378fab1332a>
    - updated to r44p1 in 2023.12
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/f1dc725659b6f4a19a18130d8f8c6de9448bca89>
  - 6.6
    - added r40p0 to `drivers/gpu/arm/mali` and added
      `drivers/gpu/arm/mali/platform/mediatek` in 2023.11
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/78229ec3056a11125c6c5de2abcd4b9b7dae3b07>
    - updated to r44p1 in 2024.03
      - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/f41ef27e3eaaaedd2d3b64e8fc053ec951640587>
- `CONFIG_MALI_PLATFORM_NAME` decides the platform used by kbase
  - it includes `platform/$(MALI_PLATFORM_DIR)/Kbuild` to build all
    platform-specific code
- the platform api is vaguely defined
  - if legacy `!CONFIG_OF`,
    - `kbase_platform_register` is called before `platform_driver_register`
    - `kbase_platform_unregister` is called after `platform_driver_unregister`
  - if legacy `CONFIG_MALI_DVFS`, `kbase_pm_get_dvfs_action` calls
    `kbase_platform_dvfs_event`
  - `get_clk_rate_trace_callbacks` defaults to `CLK_RATE_TRACE_OPS`
  - `kbasep_platform_device_*` defaults to `PLATFORM_FUNCS`
  - if `!MALI_USE_CSF`, `kbasep_platform_{context,event}_*` also defaults to
    `PLATFORM_FUNCS`
  - `kbase_pm_runtime_init` and `kbase_pm_register_access_enable` default to
    `POWER_MANAGEMENT_CALLBACKS`
- cros patches the platform api
  - <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/2f938fe02a0ee350010831313b58c32bd9027cb7>
  - the platform driver `of_match_table ` is set to platform-defined
    `kbase_dt_ids`
  - there is `kbdev->funcs` initialized from platform-defined match data, or
    `POWER_MANAGEMENT_CALLBACKS` and `PLATFORM_FUNCS`
- mediatek platform
  - `kbase_platform_register` and `kbase_platform_unregister` are nop
  - there is no `kbase_platform_dvfs_event`
  - `CLK_RATE_TRACE_OPS` uses the generic one provided by arm
  - `PLATFORM_FUNCS` and `POWER_MANAGEMENT_CALLBACKS` are NULL
    - it uses OF match data instead
- `mt8195_platform_funcs` is the real `PLATFORM_FUNCS` for MT8195
  - `platform_init_func` is set to `platform_init`
  - `kbdev->platform_context` points to `mt8195_platform_context`
  - `kbase_pm_domain_init` inits `kbdev->pm_domain_devs` and
    `kbdev->num_pm_domains`
    - this is cros-specific and should be moved to `mt8195_platform_context`
      instead...
  - `mtk_mfgsys_init` calls `devm_clk_bulk_get` to get clks, calls
    `regulator_set_voltage` to set voltages, and `mtk_map_mfg_base` to iomap
    `mediatek,mt8195-mfgcfg`
  - `mtk_set_frequency` updates the gpu freq
  - `kbdev->devfreq_ops` are updated for devfreq
    - this is cros-specific again
- `mtk_pm_callbacks` is the real `POWER_MANAGEMENT_CALLBACKS` for all socs
  - `kbase_pm_callback_power_on`
    - `regulator_enable` enables all regulators
    - `pm_runtime_get_sync` enables all pm domains
    - `clk_bulk_prepare_enable` enables all clks
    - `mtk_enable_timestamp_register` enables timestamp
  - `kbase_pm_callback_power_off`
    - `mtk_check_bus_idle` waits for idle
    - `clk_bulk_disable_unprepare` disables all clks
    - `pm_runtime_put_autosuspend` suspends all pm domains
    - `regulator_disable` disables all regulators
