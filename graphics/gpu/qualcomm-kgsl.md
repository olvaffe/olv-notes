# KGSL

## Repo

- <https://github.com/qualcomm-linux/kgsl.git>

## Configs

- `CONFIG_QCOM_KGSL=m` enables `msm_kgsl.ko`
- `CONFIG_QCOM_ADRENO_DEFAULT_GOVERNOR="msm-adreno-tz"`
  - on gen7, hw does not support `ADRENO_GMU_BASED_DCVS`. `kgsl_pwrscale_init`
    defaults to tz-fw-based `msm-adreno-tz` than `simple_ondemand` governor
- `CONFIG_QCOM_KGSL_IOCOHERENCY_DEFAULT=y`
  - on gen7, hw supports `ADRENO_IOCOHERENT` and this tells the hw to use
    coherent gpu mappings
- `CONFIG_QCOM_KGSL_IDLE_TIMEOUT=80` affects the cpu idle timer
  - on gen7, `gmu_idle_timer -> gmu_idle_check -> gen7_power_off` powers off hw
- `CONFIG_QCOM_KGSL_CONTEXT_DEBUG=y` logs debug info when a process has too
  many contexts
- `CONFIG_QCOM_KGSL_SORT_POOL=y` sorts cached pages by paddrs in the pool
  - pages are added to `pool->pool_rbtree` instead of `pool->page_list`, such
    that pages with adjacent paddrs are returned instead of simple fifo
- `CONFIG_QCOM_KGSL_USE_SHMEM=y` disables pool and uses shmem for allocations
  - `kgsl_alloc_page` calls `shmem_read_mapping_page_gfp` instead of
    `kgsl_pool_alloc_page`
- `CONFIG_QCOM_KGSL_PROCESS_RECLAIM=y` depends on shmem and supports reclaim
  - it registers a shrinker
- `CONFIG_QCOM_KGSL_SYNX=y`
  - `kgsl_hw_fence_create` calls `synx_create` instead of
    `msm_hw_fence_create` for fence
- `CONFIG_QCOM_KGSL_RT_MUTEX=y`
  - `kgsl_mutex_init` expands to `rt_mutex_init` instead of `mutex_init`, to
    avoid priority inversion
- `CONFIG_QCOM_KGSL_DEVCOREDUMP=y`
  - `kgsl_device_snapshot -> kgsl_snapshot_save_frozen_objs` calls
    `kgsl_devcoredump`
- on android, it appears that shmem and pool can co-exist
  - `kgsl_alloc_page` always allocs from the pool
  - on reclaim, it can migrate the page from the pool to shmem for swap out

## `adreno_gpu_core_gen7_5_0`

- `.features`
  - `ADRENO_APRIV` (address privilege) protects ringbuffers
  - `ADRENO_IOCOHERENT` supports coherent memory
  - `ADRENO_IFPC` (Intra-Frame Power Collapse) allows gpu to sleep between draws
  - `ADRENO_PREEMPTION` allows high-prio job to preempt low-prio job
  - `ADRENO_L3_VOTE` votes on l3 freq
  - `ADRENO_DMS` (Dynamic Mode Switching) allows gmu to switch gpu modes
    dynamically for power saving
  - `ADRENO_LPAC` (Low Power Async Compute) is separate compute queues
- `.gpudev` is `adreno_gen7_gmu_gpudev`
  - it uses swsched (`adreno_dispatch.c`), unlike `adreno_gen7_hwsched_gpudev`
- `.sqefw_name` is `gen70500_sqe.fw`
  - Software Queue Engine fw runs on CP to parse and execute PM4 packets
- `.gmufw_name` is `gen70500_gmu.bin`
  - it runs on GMU to handle power, clock, thermal, IFPC, etc.
  - it provides HFI (Host-Firmware Interface) such that KMD can communicate
    - DCVS (Dynamic Clock and Voltage Scaling) is DVFS (Dynamic Voltage and
      Frequency Scaling)
    - `hfi_gx_bw_perf_vote_cmd` requests new freq/voltage and memory bw
- `.zap_name` is `gen70500_zap.mbn`
  - tz loads and authenticates the zap shader

## Initialization

- `kgsl_3d_init` is the module entrypoint
  - `kgsl_core_init` registers `kgsl_fops`
  - `gmu_core_register` registers `a6xx_gmu_driver`
  - itself registers `adreno_platform_driver`
- `adreno_bind` binds the driver to the device
  - I think a618 uses `adreno_gpu_core_a630v2` and `adreno_a630_gpudev`
  - `a6xx_gmu_device_probe` is called
  - `a630_gmu_power_ops` is the power ops
- when `/dev/kgsl-3d0` is opened,
  - `kgsl_open` calls `kgsl_open_device` which calls `adreno_first_open` in
    `adreno_functable::first_open`
  - `adreno_first_open` calls `a6xx_gmu_first_open` in
    `a630_gmu_power_ops::first_open`
    - this is where `a630_sqe.fw` and `a630_gmu.bin` firmwares are loaded
    - `a6xx_gmu_first_boot` starts gmu and hfi
    - `a6xx_gpu_boot` starts gpu
      - `a6xx_start`
        - what is ROQ?
      - `a6xx_rb_start`
        - this sets up ringbuffer and sends `CP_ME_INIT`

## ioctls

- device
  - `IOCTL_KGSL_DEVICE_GETPROPERTY` and `kgsl_ioctl_device_getproperty`
    - `struct kgsl_device_getproperty`
      - `KGSL_PROP_DEVICE_INFO`: `chip_id`, `gmem_sizebytes`, etc.
      - `KGSL_PROP_GPU_RESET_STAT`: reset stat for drawctx for hang check
      - `KGSL_PROP_UCHE_GMEM_VADDR`: gmem iova
      - `KGSL_PROP_HIGHEST_BANK_BIT`: ubwc config
      - `KGSL_PROP_UBWC_MODE`: ubwc version
      - `KGSL_PROP_UCHE_TRAP_BASE`: always-on counter for shader clock
      - `KGSL_PROP_GPU_VA64_SIZE`: va size
      - `KGSL_PROP_IS_RAYTRACING_ENABLED`: rt
      - `KGSL_PROP_DEVICE_SHADOW`: mmap offset for drawctx timestamps
      - `KGSL_PROP_DEVICE_QTIMER`: better than `KGSL_PROP_UCHE_TRAP_BASE`
      - `KGSL_PROP_*SECURE*`: protected mem
      - `KGSL_PROP_SPEED_BIN`: speedbin
      - `KGSL_PROP_GAMING_BIN`: `gaming_bin` fuse cell
      - `KGSL_PROP_GPU_MODEL`: dt device name
      - `KGSL_PROP_VK_DEVICE_ID`: dt device id
      - `KGSL_PROP_IS_LPAC_ENABLED`: async compute queues
      - more
  - `IOCTL_KGSL_SETPROPERTY` and `kgsl_ioctl_device_setproperty`
    - `struct kgsl_device_getproperty`
      - `KGSL_PROP_PWRCTRL`: global slumber/dvfs disable, lock to pwr level 0 (peak)
      - `KGSL_PROP_PWR_CONSTRAINT`: per-drawctx constraint to specific pwr level
      - `KGSL_PROP_DCVS_PROFILE`: dvfs profiles
  - `IOCTL_KGSL_GET_FAULT_REPORT` and `kgsl_ioctl_get_fault_report`
    - `struct kgsl_fault_report`, for `VK_KHR_device_fault`
- drawctx
  - per-VkQueue
  - `IOCTL_KGSL_DRAWCTXT_CREATE` and `kgsl_ioctl_drawctxt_create`
    - `struct kgsl_drawctxt_create`
      - `KGSL_CONTEXT_SAVE_GMEM`: ignored
      - `KGSL_CONTEXT_NO_GMEM_ALLOC`: required, skip shadow gmem
      - `KGSL_CONTEXT_CTX_SWITCH`: force ctx switch even when current, for debugging
      - `KGSL_CONTEXT_PREAMBLE`: required, userspace inits gpu state in each cmdbuf
      - `KGSL_CONTEXT_PER_CONTEXT_TS`: always on, per-drawctx seqnos in memstore
      - `KGSL_CONTEXT_USER_GENERATED_TS`: drawctx seqnos are user-providied
      - `KGSL_CONTEXT_NO_FAULT_TOLERANCE`: disable fault tolerance (no transparent skip nor replay)
      - `KGSL_CONTEXT_PWR_CONSTRAINT`: per-drawctx pwr constraint
      - `KGSL_CONTEXT_PRIORITY_MASK`: prio 1 (high) to 15 (low), default to 8
      - `KGSL_CONTEXT_IFH_NOP`: skip all cmds, for benchmarking cpu overhead
      - `KGSL_CONTEXT_SECURE`: protected
      - `KGSL_CONTEXT_NO_SNAPSHOT`: no devcoredump
      - `KGSL_CONTEXT_PREEMPT_STYLE_MASK`: default, ringbuffer, finegrain
      - `KGSL_CONTEXT_TYPE_MASK`: vk, gl, etc. for debugging
      - `KGSL_CONTEXT_INVALIDATE_ON_FAULT`: disable transparent skip on fault
      - `KGSL_CONTEXT_LPAC`: async compute
  - `IOCTL_KGSL_DRAWCTXT_DESTROY` and `kgsl_ioctl_drawctxt_destroy`
    - `struct kgsl_drawctxt_destroy`
- submit
  - `IOCTL_KGSL_GPU_COMMAND` and `kgsl_ioctl_gpu_command`, for hw cmds
    - `struct kgsl_gpu_command`
  - `IOCTL_KGSL_GPU_AUX_COMMAND` and `kgsl_ioctl_gpu_aux_command`, for sw cmds
    - `struct kgsl_gpu_aux_command`
  - `IOCTL_KGSL_RECURRING_COMMAND` and `kgsl_ioctl_recurring_command`
    - `struct kgsl_recurring_command`
  - legacy
    - `IOCTL_KGSL_SUBMIT_COMMANDS`
    - `IOCTL_KGSL_RINGBUFFER_ISSUEIBCMDS`
- alloc
  - `IOCTL_KGSL_GPUOBJ_ALLOC` and `kgsl_ioctl_gpuobj_alloc`
    - `struct kgsl_gpuobj_alloc`
  - `IOCTL_KGSL_GPUOBJ_FREE` and `kgsl_ioctl_gpuobj_free`
    - `struct kgsl_gpuobj_free`
  - `IOCTL_KGSL_GPUOBJ_INFO` and `kgsl_ioctl_gpuobj_info`
    - `struct kgsl_gpuobj_info`
  - `IOCTL_KGSL_GPUOBJ_IMPORT` and `kgsl_ioctl_gpuobj_import`
    - `struct kgsl_gpuobj_import`
  - `IOCTL_KGSL_GPUOBJ_SYNC` and `kgsl_ioctl_gpuobj_sync`
    - `struct kgsl_gpuobj_sync`
  - `IOCTL_KGSL_GPUOBJ_SET_INFO` and `kgsl_ioctl_gpuobj_set_info`, for debug aid
    - `struct kgsl_gpuobj_set_info`
  - `IOCTL_KGSL_GPUMEM_BIND_RANGES` and `kgsl_ioctl_gpumem_bind_ranges`, for sparse
    - `struct kgsl_gpumem_bind_ranges`
  - legacy
    - `IOCTL_KGSL_GPUMEM_ALLOC` obsoleted by `IOCTL_KGSL_GPUOBJ_ALLOC`
    - `IOCTL_KGSL_GPUMEM_ALLOC_ID` obsoleted by `IOCTL_KGSL_GPUOBJ_ALLOC`
    - `IOCTL_KGSL_GPUMEM_FREE_ID` obsoleted by `IOCTL_KGSL_GPUOBJ_FREE`
    - `IOCTL_KGSL_GPUMEM_GET_INFO` obsoleted by `IOCTL_KGSL_GPUOBJ_INFO`
    - `IOCTL_KGSL_GPUMEM_SYNC_CACHE` obsoleted by `IOCTL_KGSL_GPUOBJ_SYNC`
    - `IOCTL_KGSL_GPUMEM_SYNC_CACHE_BULK` obsoleted by `IOCTL_KGSL_GPUOBJ_SYNC`
    - `IOCTL_KGSL_MAP_USER_MEM` obsoleted by `IOCTL_KGSL_GPUOBJ_IMPORT`
    - `IOCTL_KGSL_SHAREDMEM_FROM_PMEM` obsoleted by `IOCTL_KGSL_GPUOBJ_IMPORT`
    - `IOCTL_KGSL_SHAREDMEM_FREE` obsoleted by `IOCTL_KGSL_GPUOBJ_FREE`
    - `IOCTL_KGSL_SHAREDMEM_FLUSH_CACHE` obsoleted by `IOCTL_KGSL_GPUOBJ_SYNC`
    - `IOCTL_KGSL_CMDSTREAM_FREEMEMONTIMESTAMP_CTXTID` obsoleted by `IOCTL_KGSL_GPUOBJ_FREE`
- sync
  - each drawctx maintains 3 seqnos, aka timestamps
    - they are QUEUED, CONSUMED, and RETIRED
    - each submit is assigned a seqno
  - `IOCTL_KGSL_DEVICE_WAITTIMESTAMP_CTXTID` and `kgsl_ioctl_device_waittimestamp_ctxtid`, wait for RETIRED
    - `struct kgsl_device_waittimestamp_ctxtid`
  - `IOCTL_KGSL_TIMESTAMP_EVENT` and `kgsl_ioctl_timestamp_event`, create a sync fd signaled when RETIRED
    - `struct kgsl_timestamp_event`
  - legacy
    - `IOCTL_KGSL_CMDSTREAM_READTIMESTAMP_CTXTID` obsoleted by direct mmap of timestamps
    - `IOCTL_KGSL_SYNCSOURCE_CREATE` obsoleted as it works similar to the bad sw sync
    - `IOCTL_KGSL_SYNCSOURCE_DESTROY`
    - `IOCTL_KGSL_SYNCSOURCE_CREATE_FENCE`
    - `IOCTL_KGSL_SYNCSOURCE_SIGNAL_FENCE`
- timeline (similar to timeline drm syncobj; also backed by dma-fences)
  - `IOCTL_KGSL_TIMELINE_CREATE` and `kgsl_ioctl_timeline_create`
    - `struct kgsl_timeline_create`
  - `IOCTL_KGSL_TIMELINE_WAIT` and `kgsl_ioctl_timeline_wait`
    - `struct kgsl_timeline_wait`
  - `IOCTL_KGSL_TIMELINE_QUERY` and `kgsl_ioctl_timeline_query`
    - `struct kgsl_timeline_val`
  - `IOCTL_KGSL_TIMELINE_SIGNAL` and `kgsl_ioctl_timeline_signal`
    - `struct kgsl_timeline_signal`
  - `IOCTL_KGSL_TIMELINE_FENCE_GET` and `kgsl_ioctl_timeline_fence_get`
    - `struct kgsl_timeline_fence_get`
  - `IOCTL_KGSL_TIMELINE_DESTROY` and `kgsl_ioctl_timeline_destroy`
- counter
  - `IOCTL_KGSL_PERFCOUNTER_GET` and `adreno_ioctl_perfcounter_get`
    - `struct kgsl_perfcounter_get`
  - `IOCTL_KGSL_PERFCOUNTER_PUT` and `adreno_ioctl_perfcounter_put`
    - `struct kgsl_perfcounter_put`
  - `IOCTL_KGSL_PERFCOUNTER_QUERY` and `adreno_ioctl_perfcounter_query`
    - `struct kgsl_perfcounter_query`
  - `IOCTL_KGSL_PERFCOUNTER_READ` and `adreno_ioctl_perfcounter_read`
    - `struct kgsl_perfcounter_read`
  - `IOCTL_KGSL_PREEMPTIONCOUNTER_QUERY` and `adreno_ioctl_preemption_counters_query`
    - `struct kgsl_preemption_counters_query`
  - `IOCTL_KGSL_READ_CALIBRATED_TIMESTAMPS` and `adreno_ioctl_read_calibrated_ts`
    - `struct kgsl_read_calibrated_timestamps`

## Snapshots

- `a3xx_snapshot`
- `a5xx_snapshot`
- `a6xx_snapshot`
