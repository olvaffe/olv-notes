DRM panthor
===========

## Initialization

- `module_init(panthor_init)`
  - `panthor_mmu_pt_cache_init` inits page table cache
  - `panthor_cleanup_wq` is allocated
  - `panthor_driver` is registered
- `static struct platform_driver panthor_driver`
  - `panthor_probe` probes a `platform_device`
    - `devm_drm_dev_alloc` allocates a `panthor_device`
      - it embeds a `drm_device` which has a driver pointer to
        `panthor_drm_driver`
    - `panthor_device_init` initializes the device
  - it also provides `static DEFINE_RUNTIME_DEV_PM_OPS(panthor_pm_ops)` to PM
    - `panthor_device_suspend`
    - `panthor_device_resume`
- `panthor_device_init` initializes the device
  - `panthor_clk_init` inits `ptdev->clks`
  - `panthor_devfreq_init` inits `ptdev->devfreq`
  - `ptdev->iomem` is resource 0 `ioremap`ed
    - this is for all mmio access
  - `ptdev->phys_addr` is resource 0 phy addr
  - `panthor_gpu_init` inits `ptdev->gpu`
    - `panthor_request_gpu_irq` inits `ptdev->gpu->irq`
      - it is defined by `PANTHOR_IRQ_HANDLER`
      - `panthor_gpu_irq_handler` is the handler
  - `panthor_mmu_init` inits `ptdev->mmu`
    - `panthor_request_mmu_irq` inits `ptdev->mmu->irq`
      - `panthor_mmu_irq_handler` is the handler
  - `panthor_fw_init` inits `ptdev->fw`
    - `panthor_request_job_irq` inits `ptdev->fw->irq`
      - `panthor_job_irq_handler` is the handler
    - `panthor_vm_create` creates a VM for MCU
      - this is the vm for bos that will be allocated to hold fw
    - `panthor_fw_load` loads the fw
      - e.g., `arm/mali/arch10.8/mali_csffw.bin`
        - `GPU_ARCH_MAJOR` is 10
        - `GPU_ARCH_MINOR` is 8
      - the fw has multiple entries
      - `panthor_fw_load_entry` loads an entry
  - `panthor_sched_init` inits `ptdev->scheduler` and `ptdev->csif_info`
  - autosuspend is enabled with a 50ms delay
- `panthor_device_resume` resumes the device
  - `clk_prepare_enable` all `ptdev->clks`
  - `panthor_devfreq_resume` resumes `ptdev->devfreq`
  - `panthor_gpu_resume` resumes `ptdev->gpu`
  - `panthor_fw_resume` resumes `ptdev->fw`
  - `panthor_sched_resume` resumes `ptdev->scheduler`
  - set `ptdev->pm.state` to `PANTHOR_DEVICE_PM_STATE_ACTIVE`

## File Operations

- `panthor_open`
  - a `panthor_file` is allocated
  - `panthor_vm_pool_create` inits `pfile->vms`
  - `panthor_group_pool_create` inits `pfile->groups`
- `panthor_postclose` frees `pfile`
- `panthor_mmap` calls `drm_gem_mmap` for bos
  - it also allows mapping certain regs for direct mmio
  - `DRM_PANTHOR_USER_FLUSH_ID_MMIO_OFFSET` maps read-only
    `CSF_GPU_LATEST_FLUSH_ID` reg
- `DRM_IOCTL_PANTHOR_DEV_QUERY` maps to `panthor_ioctl_dev_query`
  - `DRM_PANTHOR_DEV_QUERY_GPU_INFO` queries `ptdev->gpu_info`
  - `DRM_PANTHOR_DEV_QUERY_CSIF_INFO` queries `ptdev->csif_info`
- `DRM_IOCTL_PANTHOR_VM_CREATE` maps to `panthor_ioctl_vm_create`
  - panvk creates a vm per-VkDevice
    - it expects some amount of VA is managed by userspace
      - first 3GB for 32-bit VA
      - first half for larger VA
    - it then reserves up to 4GB for userspace
  - `panthor_vm_pool_create_vm` creates a VM
    - the VA is splitted into
      - userspace region at the bottom
      - kernel region at the top
    - `panthor_vm_create` creates the VM to manage the kernel region
      - a kernel bo may be created with an explicit VA, or with an
        auto-assigned VA indicated by `PANTHOR_VM_KERNEL_AUTO_VA`
      - the "auto" range is used for auto-assignment
- `DRM_IOCTL_PANTHOR_VM_DESTROY` maps to `panthor_ioctl_vm_destroy`
- `DRM_IOCTL_PANTHOR_VM_BIND` maps to `panthor_ioctl_vm_bind`
  - panvk
    - it never sets `DRM_PANTHOR_VM_BIND_ASYNC` atm, which can be used to
      wait/signal syncobjs before/after an bind op
  - there is an array of `drm_panthor_vm_bind_op`
    - each op specifies a bo, an offset, a size, and a VA
    - `DRM_PANTHOR_VM_BIND_OP_TYPE_MAP` maps the range to the VA
    - `DRM_PANTHOR_VM_BIND_OP_TYPE_UNMAP` unmaps the range from the VA
- `DRM_IOCTL_PANTHOR_VM_GET_STATE` maps to `panthor_ioctl_vm_get_state`
  - this is for use with `DRM_PANTHOR_VM_BIND_ASYNC` to detect vm bind
    failures
- `DRM_IOCTL_PANTHOR_BO_CREATE` maps to `panthor_ioctl_bo_create`
  - this allocates a bo for userspace
  - `DRM_PANTHOR_BO_NO_MMAP` fails attempts to mmap
  - `exclusive_vm_id` indicates the bo is exclusive to a vm
    - it fails attempts to export the bo or bind the bo in another vm, making
      the bo private to the vm
- `DRM_IOCTL_PANTHOR_BO_MMAP_OFFSET` maps to `panthor_ioctl_bo_mmap_offset`
  - this calls the common `drm_gem_create_mmap_offset`
  - the offset is within `DRM_FILE_PAGE_OFFSET_START` and
    `DRM_FILE_PAGE_OFFSET_SIZE`
    - 1TB above 4GB
- `DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE` maps to `panthor_ioctl_tiler_heap_create`
  - panvk does not use this yet
  - `panthor_vm_get_heap_pool` returns (or creates on demand) the single heap
    pool for the VM
    - `pool->gpu_contexts` is a kernel bo
  - `panthor_heap_create` creates a heap
    - each chunk is a kernel bo
  - I guess this allocates buffers for use by the fw
- `DRM_IOCTL_PANTHOR_TILER_HEAP_DESTROY` maps to `panthor_ioctl_tiler_heap_destroy`
- `DRM_IOCTL_PANTHOR_GROUP_CREATE` maps to `panthor_ioctl_group_create`
  - panvk does not use this yet
  - a `panthor_group` seems to describe the scheduling params to the fw
    - which compute/frag/tiler cores should handle compute/frag/tiler jobs
    - group priority
    - an array of `panthor_queue`
  - a `panthor_queue` describes a queue
    - queue priority within the group
    - `queue->ringbuf` is a kernel bo
    - `queue->scheduler` is a `drm_gpu_scheduler`
    - `queue->entity` is a `drm_sched_entity`
- `DRM_IOCTL_PANTHOR_GROUP_DESTROY` maps to `panthor_ioctl_group_destroy`
- `DRM_IOCTL_PANTHOR_GROUP_GET_STATE` maps to `panthor_ioctl_group_get_state`
  - this is to detect gpu hangs
- `DRM_IOCTL_PANTHOR_GROUP_SUBMIT` maps to `panthor_ioctl_group_submit`
  - `drm_panthor_group_submit` has an array of `drm_panthor_queue_submit`
  - each `drm_panthor_queue_submit` has
    - `queue_index` is the queue index in the group
    - `stream_addr` and `stream_size` are the cmds to execute
    - `latest_flush`
    - `syncs` is an array of `drm_panthor_sync_op` to wait/signal syncobjs
  - `panthor_job_create` converts each `drm_panthor_queue_submit` to a
    `drm_sched_job`

## MMU

- `panthor_vm_create` allocates `vm->pgtbl_ops` of format `ARM_64_LPAE_S1`
  - this returns `io_pgtable_arm_64_lpae_s1_init_fns` as the init funcs
  - `arm_64_lpae_alloc_pgtable_s1` allocates the `io_pgtable`
    - `map_pages` is set to `arm_lpae_map_pages`
    - `unmap_pages` is set to `arm_lpae_unmap_pages`
- when gpuvm calls `panthor_gpuva_sm_step_map` to map,
  - `panthor_vm_map_pages` calls `ops->map_pages` to map
  - all the page tables live on system memory
- `panthor_vm_active` enables a VM
  - `panthor_mmu_as_enable` sets the regs
  - `transtab` is directly from `cfg->arm_lpae_s1_cfg.ttbr`
    - `virt_to_phys(page_address(vm->root_page_table))`
  - `transcfg` is often `0x420001c6`
    - bit 0..5: adr mode (e.g., `AS_TRANSCFG_ADRMODE_AARCH64_4K`)
    - bit 6..13: ina bits (e.g., `55 - 48 = 7`)
    - bit 14..21: ona bits
    - bit 22: sl concat
    - bit 24..27: ptw memattr (e.g., `AS_TRANSCFG_PTW_MEMATTR_WB`)
    - bit 28..29: ptw sh
    - bit 30: ptw ra (e.g., `AS_TRANSCFG_PTW_RA`)
    - bit 33: disable hier ap
    - bit 34: disable af fault
    - bit 35: wxn
    - bit 36: xreadable
  - `memattr` is translated from `cfg->arm_lpae_s1_cfg.mair`
    - `mair` define 8 attrs, each taking 8 bits
      - `arm_64_lpae_alloc_pgtable_s1` always picks `0xf404ff44`
      - attr 0 is for `ARM_LPAE_MAIR_ATTR_IDX_NC` and is
        `ARM_LPAE_MAIR_ATTR_NC` (0x44, Normal Memory, Innter and Outer Non-Cacheable)
      - attr 1 is for `ARM_LPAE_MAIR_ATTR_IDX_CACHE` and is
        `ARM_LPAE_MAIR_ATTR_WBRWA` (0xff, Normal Memory, Inner and Outer Write-back non-transient)
      - attr 2 is for `ARM_LPAE_MAIR_ATTR_IDX_DEV` and is
        `ARM_LPAE_MAIR_ATTR_DEVICE` (0x04, Device-nGnRE memory)
        - `G` and `nG` mean (non-)gathering
        - `R` and `nR` mean (non-)reordering
        - `E` and `nE` mean (non-)early write ack
      - attr 3 is for `ARM_LPAE_MAIR_ATTR_IDX_INC_OCACHE` and is
        `ARM_LPAE_MAIR_ATTR_INC_OWBRWA` (0xf4, Normal memory, Inner Non-Cacheable, Outer Write-back non-transient)
    - `memattr` define 8 attrs, each taking 8 bits
      - `0x9f9f9f9f9c4c9f4c`
      - bit 4..5: `MIDGARD_INNER`, `CPU_INNER`, `CPU_INNER_SHADER_COH`
      - bit 6..7: SHARED, NC, WB, FAULT
- AS regs
  - an AS config is controlled by
    - `AS_TRANSTAB_{LO,HI}(as)`
    - `AS_MEMATTR_{LO,HI}(as)`
    - `AS_TRANSCFG_{LO,HI}(as)`
  - an AS fault details are available at
    - `AS_FAULTSTATUS(as)`
    - `AS_FAULTADDRESS_{LO,HI}(as)`
    - `AS_FAULTEXTRA_{LO,HI}(as)`
  - an AS command is emitted by
    - `AS_STATUS(as)` returns `AS_STATUS_AS_ACTIVE` is there is an ongoing cmd
      - it must be checked before emitting another cmd
    - `AS_COMMAND(as)` emits a cmd
    - `AS_LOCKADDR_{LO,HI}(as)`
- AS commands
  - `AS_COMMAND_NOP` is unused
  - `AS_COMMAND_UPDATE` latches the latest transtab/memattr/transcfg
  - `AS_COMMAND_LOCK` locks a region
    - the region is spcified by `AS_LOCKADDR_{LO,HI}(as)`
    - it must be emitted before `AS_COMMAND_FLUSH_PT` and
      `AS_COMMAND_FLUSH_MEM`
    - it sounds like it specifies the region to flush
  - `AS_COMMAND_UNLOCK` is unused
  - `AS_COMMAND_FLUSH_PT` flushes L2 for page tables
    - it must be emitted after page table updates
  - `AS_COMMAND_FLUSH_MEM` flushes L2 for memory accesses?
    - it must be emitted before `AS_COMMAND_UPDATE`
- `panthor_mmu_irq_handler` handles faults
  - when a fault occurs,
    - `MMU_INT_STAT` is read to check for spurious irq
    - `MMU_INT_MASK` is written to disable the irq
    - `MMU_INT_RAWSTAT` is read for status
      - it is a bitmask of faulted ASs
    - `panthor_mmu_irq_handler` is called with status
    - `MMU_INT_CLEAR` is written to ack the irq
    - `MMU_INT_MASK` is written to enable the irq
    - all these are generated by `PANTHOR_IRQ_HANDLER`
  - `fault_status` is read from `AS_FAULTSTATUS(as)`
    - bit 0..7: `exception_type`
    - bit 8..9: `access_type`
      - `ATOMIC`
      - `EXECUTE`
      - `READ`
      - `WRITE`
    - bit 10: decoder fault or slave fault
      - decoder fault means bad page table?
      - slave fault means unmapped iova?
    - bit 16..31: `source_id`
  - `addr` is read from `AS_FAULTADDRESS_LO(as)`
  - it printks the fault
  - it masks out the irq
  - `panthor_mmu_as_disable` disables the VM
- `panthor_kernel_bo_create` creates a kernel bo
  - `drm_gem_shmem_create` creates a shmem gem object
    - unlike a userspace bo, `drm_gem_handle_create` is not called
  - `panthor_vm_alloc_va` allocs a VA for the bo
    - remember that a VM has a userspace region and a kernel region
    - this allocates the VA from the kernel region, managed by `vm->mm`
  - `panthor_vm_map_bo_range` maps the bo in the vm
    - `panthor_vm_prepare_map_op_ctx` pins the bo pages, obtains the unique
      `drm_gpuvm_bo`, and adds the bo to the gpuvm
    - `panthor_vm_exec_op` calls `drm_gpuvm_sm_map` to map with split-merge
      - this calls into `panthor_gpuvm_ops`
- `panthor_ioctl_vm_bind_sync`
  - `panthor_vm_bind_exec_sync_op`
- `panthor_ioctl_vm_bind_async` is similar to a group submit, except the job
  is a bind job
  - `panthor_vm_bind_job_create`
  - `panthor_vm_bind_job_prepare_resvs`

## Scheduler

- FW/HW limits
  - the fw exposes M slots and N queues
  - it means the fw can schedule up to M active groups, with each group having
    N queues
  - when there are less than M active groups in total, the fw can take care of
    scheduling by itself
  - otherwise, the driver must rotate the active groups
- `tick_work` schedules groups
  - it runs periodically only when there are more groups than the fw can
    handle, to rotate them
  - otherwise, it runs after triggered by external events
    - when a job is submitted and its group is inactive,
      `group_schedule_locked` is called and schedules a tick
    - job irq calls `panthor_sched_report_fw_events` for various reasons
      - e.g., when an active group becomes idle
- when all dependencies are met, `queue_run_job` is called to submit a job
  - `job->group` and `job->queue_idx` are the `panthor_group` and
    `panthor_queue` to submit to
  - `job->call_info` is the addr of the cmdstream to execute
  - a short `call_instrs` is appended to `queue->ringbuf`
    - note how it updates the seqno at `sync_addr`
  - `group->csg_id` indicates whether the group is active or not
    - if the group is active, the kernel rings a doorbell
    - otherwise, `group_schedule_locked` schedules the group
- when a job completes, `panthor_sched_report_fw_events` is called
  - `CSG_SYNC_UPDATE` is set for the group which triggers sync upd work
  - `group_sync_upd_work` signals `job->done_fence`
  - `sync_upd_work`
- in-syncobjs are handled in `panthor_submit_ctx_add_sync_deps_to_job`
  - `drm_syncobj_find_fence` finds the fence
  - `drm_sched_job_add_dependency` adds the fence to the job
  - in case a syncobj is in for one job and out for another,
    `panthor_submit_ctx_search_sync_signal` is used
- out-syncobjs are handled in 3 places
  - `panthor_submit_ctx_collect_jobs_signal_ops` calls
    `panthor_submit_ctx_get_sync_signal` to allocate a temp
    `panthor_sync_signal` for each out-syncobj
  - `panthor_submit_ctx_update_job_sync_signal_fences` sets `sig_sync->fence`
    to `job->s_fence->finished`
  - `panthor_submit_ctx_push_fences` calls `drm_syncobj_replace_fence` or
    `drm_syncobj_add_point` to add `sig_sync->fence` to the out-syncobj
  - later when the deps are met, `sched->ops->run_job` runs the job on hw and
    returns a hw fence
    - when the hw fence signals, `drm_sched_job_done_cb` is called to signal
      `job->s_fence->finished`
    - this indirection is necessary because we don't have the hw fence for
      out-syncobjs by the time the submit ioctl returns

## PM domains, Clocks, Regulators, and devfreq

- given `gpu: gpu@fb000000`
  - `assigned-clocks = <&scmi_clk SCMI_CLK_GPU>;`
  - `assigned-clock-rates = <200000000>;`
  - `power-domains = <&power RK3588_PD_GPU>;`
  - `clocks = <&cru CLK_GPU>, <&cru CLK_GPU_COREGROUP>, <&cru CLK_GPU_STACKS>;`
  - `clock-names = "core", "coregroup", "stacks";` - `clocks = <&cru CLK_GPU>;`
  - `operating-points-v2 = <&gpu_opp_table>;`
  - `mali-supply = <&vdd_gpu_s0>;`
- `platform_probe`
  - `of_clk_set_defaults` parses `assigned-clocks` and `assigned-clock-rates`,
    and sets the assigned clock to the specified rate
  - `dev_pm_domain_attach` parses `power-domains`, adds the device to the
    genpd, and turns the power on
- `panthor_clk_init`
  - `devm_clk_get`/`devm_clk_get_optional` parse `clocks` and `clock-names`,
    and get the clocks
- `panthor_devfreq_init`
  - `devm_pm_opp_set_regulators` shallow-parses `operating-points-v2`,
    allocates an opp table, and gets the regulators
    - `opp_table->config_regulators = _opp_config_regulator_single`
  - `devm_pm_opp_of_add_table` deep-parses `operating-points-v2` and populates
    the opp table
    - this also calls `_update_opp_table_clk` to get clks
  - `devfreq_recommended_opp` returns the best opp for the target freq
  - `dev_pm_opp_set_opp`
    - `_opp_config_regulator_single` sets the regulator to the opp voltage and
      enables the regulator
    - `_opp_config_clk_single` sets the clk to the opp rate
- `panthor_devfreq_target` is called by devfreq occasionally
  - `dev_pm_opp_set_rate` sets the target rate
    - when the target freq supported by the clk falls between two opps, this
      function sets the voltage according to the higher opp and sets the
      specified target rate
    - iow, opps specify the voltages required for different freqs, while the
      supported freqs are determined by the clk

## CSF Firmware

- `panthor_fw_init` is the entrypoint
- `platform_get_irq_byname` gets the irq for `job`
- `panthor_gpu_l2_power_on` powers on L2
  - it is a wrapper to `panthor_gpu_power_on`, which supports these gpu blocks
    - SHADER
    - TILER
    - L2
    - it appears that the fw depends on L2 and can control the rest
  - it checks `L2_PWRTRANS_{LO,HI}` to make sure there is no ongoing power
    transition
  - it writes to `L2_PWRON_{LO,HI}` to power on L2
  - it checks to `L2_READY_{LO,HI}` to make sure L2 is ready
  - `panthor_gpu_power_off` works similarly
- `panthor_vm_create` creates `fw->vm`
  - `kernel_va_start` and `kernel_va_size`
    - a vm is usually splitted into userspace region and kernel space region
    - since this vm is not visible to userspace, we don't really care about
      userspace region
    - in this case, it takes up the first 4GB
  - `auto_kernel_va_start` and `auto_kernel_va_size`
    - a bo whose va is `PANTHOR_VM_KERNEL_AUTO_VA` gets assigned an va from
      the "auto" region
    - in this case, the auto region is `[64MB, 128MB]`
- `panthor_fw_load` loads the fw
  - a fw has multiple entries
  - `panthor_fw_load_entry` loads an entry
    - `ehdr` is the entry header
    - `CSF_FW_BINARY_ENTRY_TYPE(ehdr)` returns the entry type
      - only `CSF_FW_BINARY_ENTRY_TYPE_IFACE` is supported
    - `CSF_FW_BINARY_ENTRY_SIZE(ehdr)` returns the entry size
    - `CSF_FW_BINARY_ENTRY_OPTIONAL` returns if the entry is optional
      - if an entry is not supported and is not optional, the loading fails
  - `panthor_fw_load_section_entry` loads a `CSF_FW_BINARY_ENTRY_TYPE_IFACE`
    - `panthor_fw_binary_section_entry_hdr` describes the entry
      - `flags`
        - `RD`, `WR`, and `EX` are access flags of the bo
        - `CACHE_MODE` is the cache mode of the bo
        - `PROT` means protected bo, which is ignored
        - `SHARED` means the bo is vmapped for cpu access
        - `ZERO` means the bo should be zeroed
      - `va.start` and `va.end` are the bo should bind to in the vm
        - the entry binds to `CSF_MCU_SHARED_REGION_START` is special and is
          saved to `fw->shared_section`
      - `data.start` and `data.end` are entry data to be copied into bo
      - it is followed by the entry name, which is usually empty
- `panthor_fw_start` boots the fw
  - the fw is considered booted after a `JOB_INT_GLOBAL_IF` irq is received
- `panthor_fw_init_ifaces`
- `panthor_fw_init_global_iface`

## Exceptions

- exceptions can be reported via
  - `AS_FAULTSTATUS` for mmu faults
  - `GPU_FAULT_STATUS` for gpu faults
  - shmem with CSF firmware for CS faults
- these are non-faults
  - `DRM_PANTHOR_EXCEPTION_OK`
  - `DRM_PANTHOR_EXCEPTION_TERMINATED`
  - `DRM_PANTHOR_EXCEPTION_KABOOM` (iterator)
  - `DRM_PANTHOR_EXCEPTION_EUREKA`
  - `DRM_PANTHOR_EXCEPTION_ACTIVE`
  - `DRM_PANTHOR_EXCEPTION_MAX_NON_FAULT`
- these are CS faults
  - `DRM_PANTHOR_EXCEPTION_CS_RES_TERM`
  - `DRM_PANTHOR_EXCEPTION_CS_CONFIG_FAULT`
  - `DRM_PANTHOR_EXCEPTION_CS_UNRECOVERABLE`
  - `DRM_PANTHOR_EXCEPTION_CS_ENDPOINT_FAULT`
  - `DRM_PANTHOR_EXCEPTION_CS_BUS_FAULT`
  - `DRM_PANTHOR_EXCEPTION_CS_INSTR_INVALID`
  - `DRM_PANTHOR_EXCEPTION_CS_CALL_STACK_OVERFLOW`
  - `DRM_PANTHOR_EXCEPTION_CS_INHERIT_FAULT`
  - `DRM_PANTHOR_EXCEPTION_CSF_FW_INTERNAL_ERROR`
  - `DRM_PANTHOR_EXCEPTION_CSF_RES_EVICTION_TIMEOUT`
- these are gpu faults
  - `DRM_PANTHOR_EXCEPTION_INSTR_INVALID_PC` (shader)
  - `DRM_PANTHOR_EXCEPTION_INSTR_INVALID_ENC` (shader)
  - `DRM_PANTHOR_EXCEPTION_INSTR_BARRIER_FAULT` (shader)
  - `DRM_PANTHOR_EXCEPTION_DATA_INVALID_FAULT`
  - `DRM_PANTHOR_EXCEPTION_TILE_RANGE_FAULT`
  - `DRM_PANTHOR_EXCEPTION_ADDR_RANGE_FAULT`
  - `DRM_PANTHOR_EXCEPTION_IMPRECISE_FAULT`
  - `DRM_PANTHOR_EXCEPTION_OOM`
  - `DRM_PANTHOR_EXCEPTION_GPU_BUS_FAULT`
  - `DRM_PANTHOR_EXCEPTION_GPU_SHAREABILITY_FAULT`
  - `DRM_PANTHOR_EXCEPTION_SYS_SHAREABILITY_FAULT`
  - `DRM_PANTHOR_EXCEPTION_GPU_CACHEABILITY_FAULT`
- these are mmu faults
  - the numeric suffices are page table levels
  - `DRM_PANTHOR_EXCEPTION_TRANSLATION_FAULT_[0-4]`
  - `DRM_PANTHOR_EXCEPTION_PERM_FAULT_[0-3]`
  - `DRM_PANTHOR_EXCEPTION_ACCESS_FLAG_[1-3]`
  - `DRM_PANTHOR_EXCEPTION_ADDR_SIZE_FAULT_IN`
  - `DRM_PANTHOR_EXCEPTION_ADDR_SIZE_FAULT_OUT[0-3]`
  - `DRM_PANTHOR_EXCEPTION_MEM_ATTR_FAULT_[0-3]`
