KGSL
====

## Repo

- find the kernel repos of the latest flagship devices, such as
- <https://github.com/OnePlusOSS/android_kernel_msm-5.10_oneplus_sm8450.git>

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
