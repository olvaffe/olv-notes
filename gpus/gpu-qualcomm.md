Qualcomm Adreno
===============

## History

- Snapdragon S1 released in 2008
  - Adreno 200
  - GLES 2.0
- Snapdragon S2 released in 2010
  - Adreno 205
- Snapdragon S3 released in 2010
  - Adreno 220
- Snapdragon S4 released in 2012
  - Adreno 320
  - GLES 3.0
- Snapdragon 800 released in 2013
  - Adreno 330
- Snapdragon 805 released in 2013
  - Adreno 420
  - GLES 3.2
  - Vulkan 1.0
- Snapdragon 810 released in 2014
  - Adreno 430
- Snapdragon 820 released in 2015
  - Adreno 530
- Snapdragon 835 released in 2016
  - Adreno 540
  - Vulkan 1.1
- Snapdragon 845 released in 2017
  - Adreno 630
- Snapdragon 855 released in 2018
  - Adreno 640
- Snapdragon 865 released in 2019
  - Adreno 650

## Architecture

- Adreno
  - a big L2 sitting in from of system memory
  - all memory accesses are via L2
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

## Overview

- "msm_drm_register" registers these drivers in order
  - "mdp5_driver" named "msm_mdp" for "qcom,mdp5" and more
    - this has been replaced by DPU since SDM845
  - "dpu_driver" named "msm_dpu" for "qcom,sdm845-dpu" and more
  - "dsi_phy_platform_driver" named "msm_dsi_phy" for "qcom,dsi-phy-10nm" and more
  - "dsi_driver" named "msm_dsi" for "qcom,mdss-dsi-ctrl" and more
  - "edp_driver" named "msm_edp" for "qcom,mdss-edp" and more
  - "msm_hdmi_phy_platform_driver" named "msm_hdmi_phy" for "qcom,hdmi-phy-8996" and more
  - "msm_hdmi_driver" named "hdmi_msm" for "qcom,hdmi-tx-8996" and more
  - "adreno_driver" named "adreno" for "qcom,adreno" and more
  - "msm_platform_driver" named "msm" for "qcom,sdm845-mdss" and more
- Device Tree sdm845.dtsi
  - "mdss: mdss@ae00000" compat "qcom,sdm845-mdss"
    - "mdss_mdp: mdp@ae01000" compat "qcom,sdm845-dpu"
    - "dsi0: dsi@ae94000" compat "qcom,mdss-dsi-ctrl"
    - "dsi0_phy: dsi-phy@ae94400" compat "qcom,dsi-phy-10nm"
    - "dsi1: dsi@ae96000" compat "qcom,mdss-dsi-ctrl"
    - "dsi1_phy: dsi-phy@ae96400" compat "qcom,dsi-phy-10nm"
  - "gpu@5000000" compat "qcom,adreno"
- Device Tree sdm845-cheza.dtsi
  - "edp_brij_i2c: &i2c3" compat "ti,sn65dsi86"
    - DSI-to-eDP bridge
 
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
