DRM msm
========

## Initialization

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
 
## ioctls

- `DRM_IOCTL_MSM_GET_PARAM`
  - `MSM_PARAM_GPU_ID` and `MSM_PARAM_CHIP_ID`
    - for newer gpus, DT describes them as `qcom,adreno-gmu-XYZ.W`
      - the 4 numbers are core, major, minor, and patch
    - gpu id is `XYZ`; newer GPUs return 0 to deprecate it
    - chip id is `XYZ.W` plus speedbin
  - `MSM_PARAM_MAX_FREQ`
    - newer gpus have `operating-points-v2` in DT to describe the freq table
    - this returns the highest value in the table
  - `MSM_PARAM_TIMESTAMP`
    - this reads `REG_A6XX_CP_ALWAYS_ON_COUNTER` register
  - `MSM_PARAM_GMEM_BASE` and `MSM_PARAM_GMEM_SIZE`
    - gmem iova base and size
    - base is 0 on newer gpus; 0x100000 on older gpus
    - size is 512K, 1M, 1.5M, etc.
  - `MSM_PARAM_PRIORITIES`
    - num of rings times num of sched priorities
    - 1x3 on newer gpus
  - `MSM_PARAM_PP_PGTABLE`
    - deprecated; always 0
  - `MSM_PARAM_FAULTS`
    - global faults (pre-a6xx) or per-context faults (post-a6xx)
      - a6xx supports per-context address spaces
    - for userspace to report gpu lost
  - `MSM_PARAM_SUSPENDS`
    - incremented whenever the gpu is powered down
    - some counters are reset after power down
      - `REG_A6XX_CP_ALWAYS_ON_COUNTER` used for gpu timestamps
      - performance counters
    - userspace would like to handle that
  - `MSM_PARAM_VA_START` and `MSM_PARAM_VA_SIZE`
    - return the iova address space
    - useful if userspace wants to manage the iova address space (for
      virtualization or for trace record/replay)
- `DRM_IOCTL_MSM_SET_PARAM`
  - `MSM_PARAM_SYSPROF`
    - only root can set this param
    - when any context sets this to 1, counter reset on context switch is
      disabled
    - when any context sets this to 2, `pm_runtime` is disabled
  - `MSM_PARAM_COMM` and `MSM_PARAM_CMDLINE`
    - by default, they are `comm` and `cmdline` of the thread that creates the
      context for debugging
    - they can be overridden and is useful for virtualization
- `DRM_IOCTL_MSM_GEM_NEW`
  - `MSM_BO_SCANOUT`
    - use the vram (carved out from system memory) on older gpus
    - no-op otherwise
  - `MSM_BO_GPU_READONLY`
    - map in iommu without `IOMMU_WRITE`
  - `MSM_BO_CACHED`
    - mapping uses default prot (cached)
    - is non-coherent
  - `MSM_BO_WC`
    - mapping uses `pgprot_writecombine`
    - abuse dma api to flush/invalidate cpu cache on alloc/free
  - `MSM_BO_CACHED_COHERENT`
    - map in iommu with `IOMMU_CACHE` (dma cache coherency)
    - require a6xx
- `DRM_IOCTL_MSM_GEM_INFO`
  - `MSM_INFO_GET_OFFSET` gets the magic offset for `mmap`
  - `MSM_INFO_GET_IOVA` gets the iova
  - `MSM_INFO_SET_NAME` sets the debug name
  - `MSM_INFO_GET_NAME` gets the debug name
  - `MSM_INFO_SET_IOVA` sets the (userspace-managed) iova
- `DRM_IOCTL_MSM_GEM_CPU_PREP`
  - calls `dma_resv_wait_timeout` to wait the `dma_resv` before cpu access
    - only implicit fencing; no cache management
  - `MSM_PREP_READ` waits for pending implicit write fences
  - `MSM_PREP_WRITE` waits for pending implicit read fences
  - `MSM_PREP_NOSYNC` tests and returns immediately
- `DRM_IOCTL_MSM_GEM_CPU_FINI`
  - nop
- `DRM_IOCTL_MSM_WAIT_FENCE`
  - wait until the specified submitqueue passes the specified fence id
- `DRM_IOCTL_MSM_GEM_MADVISE`
  - hints used by the shrinker
  - `MSM_MADV_WILLNEED` for when a bo is moved out of a userspace bo cache
  - `MSM_MADV_DONTNEED` for when a bo is moved into a userspace bo cache
- `DRM_IOCTL_MSM_SUBMITQUEUE_NEW` and `DRM_IOCTL_MSM_SUBMITQUEUE_CLOSE`
  - for gallium, a `pipe_screen` is a kernel context and a `pipe_context`
    is a submitqueue
- `DRM_IOCTL_MSM_SUBMITQUEUE_QUERY`
  - `MSM_SUBMITQUEUE_PARAM_FAULTS` for per-submitqueue faults
  - incremented on every hanged submit
- `DRM_IOCTL_MSM_GEM_SUBMIT`
  - an array of `drm_msm_gem_submit_bo`
    - `submit_lookup_objects` copies the array from userspace, massages the
      array, and looks up bos
    - `submit_lock_objects` calls `dma_resv_lock_interruptible` on all bos'
      `dma_resv` and sets `BO_LOCKED`
    - `submit_pin_objects` calls `msm_gem_get_vma_locked` and
      `msm_gem_pin_vma_locked` to pin bos and sets `BO_OBJ_PINNED` (pages
      pinned) and `BO_VMA_PINNED` (vma pinned).  It also sets `BO_VALID`
      unless the bo has been moved (never happens with softpin)
    - before returning, `submit_cleanup_bo` is called on all bos with
      `BO_LOCKED` to unlock them
      - bo pages and vma are pinned until completion
    - `MSM_SUBMIT_BO_READ` indicates the bo is read from (will add a read
      implicit fence)
    - `MSM_SUBMIT_BO_WRITE` indicates the bo is written to (will add a write
      implicit fence)
    - `MSM_SUBMIT_BO_DUMP` indicates the bo should be dumped in debug dump
  - an array of `drm_msm_gem_submit_cmd`
    - `submit_lookup_cmds` copies the array from userspace and massages the array
      - each cmd describes a region of a bo
      - `drm_msm_gem_submit_reloc` is for iova patching and is unused with
        softpin
    - `MSM_SUBMIT_CMD_BUF` is added to the ring
    - `MSM_SUBMIT_CMD_IB_TARGET_BUF` is unused because of softpin
      - used to let relocs patch IBs
    - `MSM_SUBMIT_CMD_CTX_RESTORE_BUF` is added to the ring only if there is a
      context switch; unused
  - an in array of `drm_msm_gem_submit_syncobj`
    - `MSM_SUBMIT_SYNCOBJ_RESET`
  - an out array of `drm_msm_gem_submit_syncobj`
  - `flags`
    - `MSM_SUBMIT_NO_IMPLICIT` ignores implicit in-fences
      - without this flag, the submit job respects implicit in-fences
        - if the submit reads a bo and the bo has implicit write in-fences,
          the submit must wait to avoid RAW hazard
        - if the submit writes a bo and the bo has implicit read in-fences,
          the submit must wait to avoid WAR hazard
      - with this flag, the submit job ignores implicit in-fences
        - There was a time when there can only be a single implicit write
          fence.  If the submit writes to a bo and adds its own implicit write
          fence, it needs to wait on the implicit write in-fence before
          replacing it.  This should have been fixed.
      - note that this flag does not affect implicit out-fences
        - each submit is always associated with a fence (`submit->user_fence`)
          and a fence id (`submit->fence_id` and `queue->last_fence`, but not
          to be confused with fence context seqno)
        - `submit_attach_object_fences` always adds the fence as an implicit
          out-fence to each bo
    - `MSM_SUBMIT_FENCE_FD_IN`
      - call `drm_sched_job_add_dependency` to add `fence_fd` as an extra
        dependency of the job
    - `MSM_SUBMIT_FENCE_FD_OUT`
      - return a sync fd in `fence_fd`, pointing to the submit's fence
    - `MSM_SUBMIT_SUDO` is deprecated
      - when set, cmdstreams are copied into the ring instead of indirectly
        executed
    - `MSM_SUBMIT_FENCE_SN_IN`
      - without the flag, the fence id is assigned by kernel.  The id can be
        used with `DRM_IOCTL_MSM_WAIT_FENCE`
      - with the flag, the fence id is specified by the userspace.
    - `MSM_SUBMIT_SYNCOBJ_IN`
      - `msm_parse_deps` calls `drm_sched_job_add_dependency` to add the
        underlying fences of the syncobjs as extra job dependencies
      - `MSM_SUBMIT_SYNCOBJ_RESET` resets the syncobjs (removing
        `syncobj->fence` which makes them unsignaled)
    - `MSM_SUBMIT_SYNCOBJ_OUT`
      - `msm_parse_post_deps` looks up the syncobjs
      - `msm_process_post_deps` adds the submit's fence into the syncobjs

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

## GPU

- When device "qcom,adreno" is bound,
  - `adreno_info` is determined
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
  - init is called to get an `msm_gpu`
    `- a6xx_gpu_init` calls `adreno_gpu_init` with 1 ring
    `- msm_gpu_init` creates a 4G address space as requested
    - it also creates 1 `msm_ringbuffer` as requested
      - a 32K BO
      - start/end/enxt/cur as indices
      - submits is a list of ?
      - fctx is a fence context goes by the name "gpu-ring-%d"
      - memptrs points to a BO managed by msm_gpu
  - static const `struct adreno_gpu_funcs` funcs
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
- when the DRM device is opened, `adreno_load_gpu` is called
  - it loads the GPU firmware
  - turns on the PM
  - calls `msm_gpu_hw_init`

## MDSS

- componentized device
  - when a subdevice is probed, add it with `component_add`
  - when the master device is probed, describe the required subdevices and
    add the master with `component_master_add_with_match`
    - it will attempt to bind the master device to its driver
    - the driver init function calls `component_bind_all` to bind the
      subdevices to their respective drivers
- `msm_drm_init` is the driver init function for the master device
  - it initializes MDSS with `dpu_mdss_init` first
  - it then binds all subdevices
  - the binding of DPU initializes KMS with `dpu_bind`
  - it then calls kms's `hw_init`
  - it starts a `crtc_commit:%d` and a `crtc_event:%d` thread for each crtc
  - `drm_vblank_init`

## Memory Management

- ioctls
  - `MSM_GEM_NEW`
    - `msm_ioctl_gem_new` takes a size and a flags
       - it kzallocs `a msm_gem_object`, which is also a `drm_gem_object`
       - the gem object is added to the device's private `inactive_list`
       - `drm_gem_object_init` is called to set up filp
       - `drm_gem_handle_create` is called and the handle is the id into file's private `object_idr`
  - `DRM_IOCTL_GEM_CLOSE`
    - `drm_gem_handle_delete` is called and it deletes the handle in file's private `object_idr`
    `- drm_gem_object_release_handle` is called
      - if there are prime handles, they are removed from file's private prime table
      - if there are vmas with this file's tag, they are removed
      - if there is a name, it is removed from the device's `object_name_idr`
      - if exported as `dma_buf`, the handle is removed
  - `DRM_IOCTL_GEM_FLINK`
    - it creates a name for the BO and adds the name to device's `object_name_idr`
  - `DRM_IOCTL_GEM_OPEN`
    - it looks up a BO using its name, and create a handle to the file's `object_idr`
  - `DRM_IOCTL_PRIME_HANDLE_TO_FD`
    - it looks up a BO from the file's `object_idr`
    - it calls `drm_gem_prime_export` and save the `dma_buf` to BO's `dma_buf`
    - it inserts the `dma_buf` into file's prime table
    - it finds an fd for the `dma_buf` and return the fd
    - if the BO was imported, use the original `dma_buf`
    - if the BO has been exported, use the same `dma_buf`
  - `DRM_IOCTL_PRIME_FD_TO_HANDLE`
    - if the dma-buf was already imported or exported, return
    - if the dma-buf was created by the device, grab the backing BO
    - if the dma-buf points to a non-GEM BO, create a new BO around its pages
    - create a handle for the BO in file's `object_idr`
    - add the `dma_buf` to the file's prime table
  - `MSM_GEM_INFO`
    - when `MSM_INFO_IOVA` is set, return the offset in GPU `address_space`
      - create a new VMA
      - `get_pages()` to get the pages and sg table
      - `msm_gem_map_vma` to initialize the VMA and to set up IOMMU
      - return the offset of the VMA
    - otherwise, intialize `obj->vma_node` within `dev->vma_offset_manager` and
      returns the VMA offset
  - `MSM_GEM_MADVISE`
  - `MSM_GEM_CPU_PREP`
  - `MSM_GEM_CPU_FINI`
- mmap
  - use the VMA to look up the object from `dev->vma_offset_manager`
  `- drm_vma_node_is_allowed` checks if the object has an associlated file private handle
  - set `vma->vm_ops` to driver's `vm_operations_struct`
  - in fault handler, `get_pages()` and update MMU
- `reservation_object`
  - each newly created BO owns a `reservation_object`
  - each dma-buf import creates a BO with the `reservation_object` of the dma-buf
  - a `reservation_object` can hold an exclusive `dma_fence` and a list of shared `dma_fence`

## Command Submission and Fence

- Fence
  - `msm_fence_context` is a seqno-based per-ring fence context
    - it goes by the name "gpu-ring-%d"
    - each `dma_fence` allocated from the context is assocaited with a seqno (`++fctx->last_fence`)
    - each bo submission is assocaited with a `dma_fence` (and its seqno)
    - when a submission completes, `msm_update_fence` is called to update
      fctx->completed_fence to the submission's seqno.
    - to check if a `dma_fence` is signaled, (`fctx->completed_fence >= fence->seqno`)
  - each submission may have a fence fd input
    - it must point a valid `sync_file` if not -1
    - `sync_file_get_fence` converts it to a `dma_fence`
    - if the `dma_fence` is not from `ring->fctx`, do a blocking
      `dma_fence_wait`
  - each submission may request a fence fd output
    - it converts the `dma_fence` associated with the submission into a `sync_file`
    - and return the fd to the userspace
- `MSM_GEM_SUBMIT`
  - a submission consists of a bunch of `drm_msm_gem_submit_cmd` and `drm_msm_gem_submit_bo`
    - each bo has a flag, indicating
  - `submit_lookup_objects` initializes `submit->bos` from `drm_msm_gem_submit_bo` array
    - it looks up handles from `file->object_idr` to get BOs
    - it temporarily sets iova of each BOs to the presumed address set by userspace
  - `submit_lock_objects` locks the BOs' `reservation_object`
    - if failed to lock any BO, it does a slow lock on the BO
  - `submit_fence_sync` waits for the BOs to be ready
    - if the BO is only read-only by the submission, reserve a shared fence
      for the BO.  The shared fence makes sure no one writes to the BO before
      the submission completes.
    - return early if the explicit fence fd input is used
    - otherwise, `msm_gem_sync_object` is called
      - if there is a explicit fence for the BO (someone wrote to the BO), do
        a CPU wait with `dma_fence_wait`
      - if we are going to write to the BO, do CPU wait on all shared fences
        of the BO
  - `submit_pin_objects` pins all BOs and finalizes their IOVAs
    - if presumed address is correct, mark `BO_VALID`
    - otherwise, clear `BO_VALID`.  Need to apply reloc.
  - for each `drm_msm_gem_submit_cmd`,
    - lookup the BO (which is specified using `drm_msm_gem_submit_bo`)
    - remember the type, offseted IOVA, size
    - if reloc is required, `submit_reloc` applies the reloc list
  - `msm_gpu_submit` is called
    - it calls `msm_gpu_hw_init` in case the GPU was suspended
    - `submit->seqno = ++ring->seqno`, which the ring's seqno, unrelated to fence seqno
    - it dumps the submission with `msm_rd_dump_submit`, if the debugfs file is opened
    - it updates GPU SW counters
    - for each BO, if marked for reading or writing, call `msm_gem_move_to_active`
      - this adds the submission's `dma_fence` to the `reservation_object` of the BO
      - also move the BO from drm `inactive_list` to gpu `active_list`
    - `a6xx_submit` is called
      - for each cmd,
        - emit `CP_INDIRECT_BUFFER_PFE` with the cmd's IOVA offset and size
      - emit `REG_A6XX_CP_SCRATCH_REG` to write submit->seqno (ring seqno) to scratch reg
      - emit `CP_EVENT_WRITE`
        - arg1: request timestamp and an interrupt
        - arg2: address of msm_rbmemptrs.fence
        - arg3: submit->seqno
        - this writes submit->seqno to msm_rbmemptrs.fence
      - write to `REG_A6XX_CP_RB_WPTR` to update write pointer
    - there are three types of commands
      - `MSM_SUBMIT_CMD_IB_TARGET_BUF`: emit `CP_INDIRECT_BUFFER_PFE` to execute
      - `MSM_SUBMIT_CMD_CTX_RESTORE_BUF`: emit `CP_INDIRECT_BUFFER_PFE` to execute
        _if_ the last submit is from a different opened drm file
      - `MSM_SUBMIT_CMD_IB_TARGET_BUF`: ignored (except reloc was applied in a prior step)
  - `msm_gpu_retire` is triggered by the irq requested by `msm_gpu_submit` 
    - it wakes up `retire_worker`
      - `update_fences` calls `msm_update_fence` with all completed submission fences
      - for each completed submission, its `msm_gem_submit` is freed
    - it updates SW counters
- other ioctls
  - `MSM_SUBMITQUEUE_NEW`
    - create a file-local `msm_submitqueue` for a ring buffer
  - `MSM_SUBMITQUEUE_CLOSE`
    - destroy a `msm_submitqueue`
  - `MSM_WAIT_FENCE` is legacy path and calls `msm_wait_fence` to wait for a specified seqno

## debugfs

- dma-buf
  - `/d/dma_buf/buf_info`
    - `dma_buf_export` creates a new `dma_buf`.  This file shows all `dma_buf`,
      who did the exports, exclusive and shared fences,
- `drm_core_init` adds
  - `/d/dri/drm_master_relax` allows normal users to set/drop master
- `drm_debugfs_init` adds
  - `/d/dri/<minor>/name` dumps dev name
  - `/d/dri/<minor>/clients` dumps all opened files of the dev
  - `/d/dri/<minor>/gem_names` dumps `object_name_idr` (flink names)
- `drm_atomic_debugfs_init` adds
  - `/d/dri/<minor>/state`, enabled when atomic commit is supported, print the plane/crtc/connector states
- `drm_framebuffer_debugfs_init` adds
  - `/d/dri/<minor>/framebuffer` lists the current framebuffers
- `drm_client_debugfs_init` adds
  - `/d/dri/<minor>/internal_clients` lists the in-kernel clients, which is only used by fbdev
- `drm_debugfs_crtc_add` adds
  - `/d/dri/<minor>/crtc-<index>`
- `drm_debugfs_connector_add` adds
  - `/d/dri/<minor>/<connector-name>/force` to force connector on/off
  - `/d/dri/<minor>/<connector-name>/edid_override` for EDID override
- `msm_debugfs_init` adds
  - `/d/dri/<minor>/gem` dumps all GEM objects
  - `/d/dri/<minor>/mm` dumps `vma_offset_manager` mm
  - `/d/dri/<minor>/fb` dumps all framebuffers
  - `/d/dri/<minor>/gpu` dumps GPU states (crash the system!)
  - `/d/dri/<minor>/kms` dumps DPU states
  - `/d/dri/<minor>/hangcheck_period_ms` sets hangcheck timeout (default 500ms)
    - `msm_gpu_submit` calls `hangcheck_timer_reset` to reset the timer
    - when the timer fires, `hangcheck_fence` checks if fence seqno has been
      updated
- `dpu_kms_debugfs_init` adds
  - `/d/dri/<minor>/debug/*`, a bunch of stuff
- `msm_rd_debugfs_init` adds
  - `/d/dri/<minor>/rd`, when opened, dump all submits
  - `/d/dri/<minor>/hangrd`, when opened, dump the submit that triggers hangchecker
- `msm_perf_debugfs_init` adds
  - `/d/dri/<minor>/perf`, when opened, starts SW and hW counters and dumps the stats
- `_dpu_crtc_init_debugfs` adds
  - `/d/dri/<minor>/crtc<id>`
- `_dpu_encoder_init_debugfs` adds
  - `/d/dri/<minor>/encoder<id>`
- `_dpu_plane_init_debugfs` adds
  - `/d/dri/<minor>/plane<id>`
   
## cffdump

- an rd dump consists of sections
  - each section has
    - 4-byte type
    - 4-byte payload size
    - payload
  - section types
    - `RD_GPU_ID` for gpu id
    - `RD_CHIP_ID` for chip id
    - `RD_CMD` for comm
    - more
- `rd_open` is called when userspace opens the rd file
  - it writes `RD_GPU_ID` and `RD_CHIP_ID` on open
- `msm_rd_dump_submit` is called for each submit
  - if rd is not open, it early returns
  - it writes `RD_CMD` for comm name, comm pid, submit ring seqno
  - for each bo in the submit, `snapshot_buf` is called
    - it always writes `RD_GPUADDR` for iova
    - if the bo is to be fully dumped, it writes `RD_BUFFER_CONTENTS` for
      contents
      - one can set `rd_full` to force the behavior
  - for each bo that contains cmdstream, it writes `RD_CMDSTREAM_ADDR` for the
    cmdstream offset/size

## GPU Hang

- IRQs
  - looking for `request_irq`, msm asks for a few irqs
  - `a6xx_hfi_irq` for hardware firmware interface, a queue that cpu uses to
    communicate with gmu
  - `a6xx_gmu_irq` for graphics management unit, for gpu power management
  - `a6xx_irq` for gpu core
  - `dpu_core_irq` for display processing unit
  - `a6xx_fault_handler` for iommu fault handler
    - adreno is behind `qcom,adreno-smmu` iommu
- `msm_gpu_crashstate_capture`
  - it can be called by `recover_worker` on adreno fault or by `fault_worker`
    on iommu fault
  - `a6xx_gpu_state_get` collects important gpu state info `msm_gpu_state`
  - `msm_gpu_crashstate_get_bo` add dumpable submit bos into `msm_gpu_state`
  - it also calls `dev_coredumpm` to integrate with dev coredump
