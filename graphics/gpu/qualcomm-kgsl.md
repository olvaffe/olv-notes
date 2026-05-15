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

## Snapshots

- `a3xx_snapshot`
- `a5xx_snapshot`
- `a6xx_snapshot`
