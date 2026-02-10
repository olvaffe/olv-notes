DRM panthor
===========

## Initialization

- `module_init(panthor_init)`
  - `panthor_mmu_pt_cache_init` creates `kmem_cache` for page tables
  - `alloc_workqueue` allocs `panthor_cleanup_wq`
  - `platform_driver_register` registers `panthor_driver`
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
  - `ptdev->pm` is initialized
    - `state` is `PANTHOR_DEVICE_PM_STATE_SUSPENDED`
  - `ptdev->reset` is initialized
    - `work` is `panthor_device_reset_work` to reset gpu on errors
    - `wq` is for reset work as well as fw ping work
    - `fast` is for fw suspend/resume, unrelated to reset
  - `panthor_clk_init` inits `ptdev->clks`
  - `panthor_devfreq_init` inits `ptdev->devfreq`
  - `ptdev->iomem` is resource 0 `ioremap`ed
    - this is for all mmio access
    - page 0 is `GPU_CONTROL`
    - page 1 is `JOB_CONTROL`
    - page 2 is `MMU_CONTROL`
  - `ptdev->phys_addr` is resource 0 phy addr
  - `devm_pm_runtime_enable` enables runtime pm
    - `device_pm_init` sets `dev->power.disable_depth` to 1
    - this decrements `dev->power.disable_depth` to 0
  - `pm_runtime_resume_and_get` resumes the device and increments usage count
    - it calls `panthor_device_resume` to power on the device for mmio access
  - `panthor_gpu_init` inits `ptdev->gpu`
    - `panthor_request_gpu_irq` inits `ptdev->gpu->irq`
      - it is defined by `PANTHOR_IRQ_HANDLER`
      - `panthor_gpu_irq_handler` is the handler
  - `panthor_mmu_init` inits `ptdev->mmu`
    - `panthor_request_mmu_irq` inits `ptdev->mmu->irq`
      - `panthor_mmu_irq_handler` is the handler
  - `panthor_fw_init` inits `ptdev->fw` and part of `ptdev->csif_info`
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
  - `panthor_sched_init` inits `ptdev->scheduler`, `ptdev->fw_info`, and
    `ptdev->csif_info`
  - `pm_runtime_use_autosuspend` enables autosuspend
    - `pm_runtime_set_autosuspend_delay` sets the delay to 50ms
  - `drm_dev_register` registers the dev
  - `pm_runtime_put_autosuspend` decrements the usage count and starts the
    autosuspend timer

## PM: PM domains, Clocks, Regulators, and devfreq

- given `gpu: gpu@fb000000`
  - `assigned-clocks = <&scmi_clk SCMI_CLK_GPU>;`
  - `assigned-clock-rates = <200000000>;`
  - `power-domains = <&power RK3588_PD_GPU>;`
  - `clocks = <&cru CLK_GPU>, <&cru CLK_GPU_COREGROUP>, <&cru CLK_GPU_STACKS>;`
  - `clock-names = "core", "coregroup", "stacks";`
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
    - `opp_table->config_clks = _opp_config_clk_single`
  - `devfreq_recommended_opp` returns the closest opp for the current freq
  - `dev_pm_opp_set_opp` sets the opp
    - `_opp_config_regulator_single` sets the regulator to the opp voltage and
      enables the regulator
    - `_opp_config_clk_single` sets the clk to the opp rate
  - `ptdev->fast_rate` is set to the highest opp freq
  - `devm_devfreq_add_device` creates devfreq dev
    - every 50ms, `devfreq_monitor` calls `devfreq_simple_ondemand_func` to
      determine the next freq and calls `devfreq_set_target` to set the freq
    - `panthor_devfreq_get_dev_status` is called from `devfreq_update_stats`
      to update utilization
    - `panthor_devfreq_target` calls `dev_pm_opp_set_rate` to set the freq
      - when the target freq supported by the clk falls between two opps, this
        function sets the voltage according to the higher opp and sets the
        specified target rate
      - iow, opps specify the voltages required for different freqs, while the
        supported freqs are determined by the clk
  - `devfreq_cooling_em_register` registers the devfreq dev for cooling
    (thermal throttling)
- `panthor_device_resume` resumes the device
  - `clk_prepare_enable` preps and enables each clk in `ptdev->clks`
  - `panthor_devfreq_resume` resumes `ptdev->devfreq`
  - `panthor_gpu_resume` resumes `ptdev->gpu`
  - `panthor_mmu_resume` resumes `ptdev->mmu`
  - `panthor_fw_resume` resumes `ptdev->fw`
  - `panthor_sched_resume` resumes `ptdev->scheduler`
  - set `ptdev->pm.state` to `PANTHOR_DEVICE_PM_STATE_ACTIVE`
- `panthor_device_suspend` suspends the device
  - `panthor_sched_suspend` suspends `ptdev->scheduler`
  - `panthor_fw_suspend` suspends `ptdev->fw`
  - `panthor_mmu_suspend` suspends `ptdev->mmu`
  - `panthor_gpu_suspend` suspends `ptdev->gpu`
  - `panthor_devfreq_suspend` suspends `ptdev->devfreq`
  - `clk_disable_unprepare` disables and unpreps each clk in `ptdev->clks`
  - set `ptdev->pm.state` to `PANTHOR_DEVICE_PM_STATE_SUSPENDED`

## IRQ

- `PANTHOR_IRQ_HANDLER` defines an irq handler
  - `panthor_request_foo_irq` allocs an irq line
  - `panthor_foo_irq_resume` is called on resume
    - it writes to `FOO_INT_CLEAR` to clear bits in `FOO_INT_RAWSTAT`
    - it writes to `FOO_INT_MASK` to enable the specified sources
  - `panthor_foo_irq_raw_handler` handles an irq
    - it reads `FOO_INT_STAT` to check for spurious irq
      - `FOO_INT_STAT` is `FOO_INT_RAWSTAT & FOO_INT_MASK`
    - it writes to `FOO_INT_MASK` to disable all sources
    - it returns `IRQ_WAKE_THREAD` to wake the threaded handler
  - `panthor_foo_irq_threaded_handler` continues irq handling
    - it reads `FOO_INT_RAWSTAT` to get irq sources
    - it masks out souces that are not enabled
    - it calls `panthor_foo_irq_handler`, which writes to `FOO_INT_CLEAR` to
      clear sources
    - it writes to `FOO_INT_MASK` to re-enable sources
  - `panthor_foo_irq_suspend` is called on suspend
    - it writes to `FOO_INT_MASK` to disable all sources
- IOW, upon irq,
  - raw handler
    - reads `FOO_INT_STAT` to detect spurious irqs
    - writes 0 to `FOO_INT_MASK` to disable sources from asserting lines
    - returns `IRQ_WAKE_THREAD`
  - threaded handler
    - reads `FOO_INT_RAWSTAT` for sources
    - loops until all sources are handled
    - writes to `FOO_INT_MASK` to re-enable sources to assert lines
- `PANTHOR_IRQ_HANDLER(gpu, GPU, panthor_gpu_irq_handler)`
  - sources are gpu faults, gpu cmd completions, etc.
- `PANTHOR_IRQ_HANDLER(mmu, MMU, panthor_mmu_irq_handler)`
  - sources are which of the 8 address spaces faults or completes `AS_COMMAND_x`
- `PANTHOR_IRQ_HANDLER(job, JOB, panthor_job_irq_handler)`
  - sources are glb and any of the csg

## GPU

- `panthor_gpu_init` inits `ptdev->gpu`
  - `reqs_acked` is waitqueue for `GPU_CMD`
  - `panthor_request_gpu_irq` requests selected sources
    - `GPU_IRQ_FAULT`
    - `GPU_IRQ_PROTM_FAULT`
    - `GPU_IRQ_RESET_COMPLETED`
    - `GPU_IRQ_CLEAN_CACHES_COMPLETED`
- `panthor_gpu_power_on` and `panthor_gpu_power_off`
  - they power one of the gpu blocks on/off
    - `L2` for L2
    - `SHADER` for shader cores
    - `TILER` for tiler
  - because of `PWROFF_HYSTERESIS_US`, `SHADER` and `TILER` are managed by CSF
    automatically
  - how they work is
    - wait for in-flight transitions until `PWRTRANS` is cleared
    - write to `PWRON` or `PWROFF`
    - wait until `READY` is set (on) or until `PWRTRANS` is cleared (off)
- `panthor_gpu_irq_handler`
  - if `GPU_IRQ_FAULT` or `GPU_IRQ_PROTM_FAULT`, it prints an error
  - otherwise, it clears `ptdev->gpu->pending_reqs` and wakes up waiters
- `panthor_gpu_flush_caches` flushes/invalidates l2/lsc/other caches
  - it is called before suspend, to ensure caches are flushed
  - it sets `ptdev->gpu->pending_reqs`
  - it writes to `GPU_CMD` for `GPU_FLUSH_CACHES`
  - it waits until `ptdev->gpu->pending_reqs` is cleared
- `panthor_gpu_soft_reset` soft-resets the gpu
  - it is called for gpu recovery, or for a slow path during suspend
  - it sets `ptdev->gpu->pending_reqs`
  - it writes to `GPU_CMD` for `GPU_SOFT_RESET`
  - it waits until `ptdev->gpu->pending_reqs` is cleared

## MMU

- `panthor_mmu_init` inits `ptdev->mmu`
  - `mmu->as.lru_list` is for VMs that are active but idle
    - when an active vm becomes idle, `panthor_vm_idle` adds the vm to the
      tail of the list withou evicting the vm
    - when the vm becomes busy again, `panthor_vm_active` removes it from the
      list and early returns
    - when another vm becomes active but there is no available slot,
      `panthor_vm_active` evicts the first vm on the list
  - `mmu->vm.list` is a list of all VMs
  - `panthor_request_mmu_irq` requests faults for all address spaces
  - `mmu->vm.wq` is a workqueue shared by all VMs' `vm->sched` for
    `panthor_vm_bind_run_job`
- `panthor_mmu_irq_handler` handles an irq
  - `status` indicates which ASs faulted
  - for each AS that faulted
    - `mmu->as.faulty_mask` is set
    - an error is logged
    - the fault is acked by writing to `MMU_INT_CLEAR`
    - the AS is no longer an irq source by updating `mmu->irq.mask`
    - `vm->unhandled_fault` is set
    - `panthor_mmu_as_disable` disables the AS
  - `panthor_sched_report_mmu_fault` reports the fault to sched to force a
    tick
- when fw or userspace creates a VM (per-`VkDevice`), `panthor_vm_create` is
  called to create a `panthor_vm`
  - the vm region is splitted into userspace region and kernel region
    - userspace region is before `kernel_va_start` and is managed by userspace
    - kernel region is specified by `kernel_va_start` and `kernel_va_size`,
      and is managed by kernel
      - a kernel bo whose va is `PANTHOR_VM_KERNEL_AUTO_VA` gets assigned an
        va from the "auto" region specified by `auto_kernel_va_start` and
        `auto_kernel_va_size`
  - `vm->mm` is a `drm_mm` to manage the kernel va range (top 2TB)
  - `vm->as.id` is -1 until the vm is active
  - `pgtbl_ops` will alloc/free page tables in `ARM_64_LPAE_S1` format
    - this returns `io_pgtable_arm_64_lpae_s1_init_fns` as the init funcs
    - `arm_64_lpae_alloc_pgtable_s1` allocates the `io_pgtable`
      - `map_pages` is set to `arm_lpae_map_pages`
      - `unmap_pages` is set to `arm_lpae_unmap_pages`
    - `alloc_pt` and `free_pt` will alloc/free page tables
  - `sched` is a `drm_gpu_scheduler` for VM ops
    - they are executed on cpu using `drm_gpuvm`
  - `entity` is a `drm_sched_entity`
  - the vm is added to `ptdev->mmu->vm.list`
  - `base` is a `drm_gpuvm` with `panthor_gpuvm_ops`
- when userspace performs a sync vm bind, `panthor_vm_bind_exec_sync_op` is
  called
  - it mainly calls the generic `drm_gpuvm_sm_map` to map and the generic
    `drm_gpuvm_sm_unmap` to unmap
    - if another bo is already mapped at the same range, the generic function
      takes care of unmapping overlapped range of the existing bo
  - `panthor_gpuvm_ops` provides the ops to the generic gpuvm
- when gpuvm calls `panthor_gpuva_sm_step_map` to map,
  - `panthor_vm_map_pages` calls `ops->map_pages` to map
  - `panthor_vm_flush_range` invalidates TLB and L2
- when gpuvm calls `panthor_gpuva_sm_step_unmap` to unmap,
  - `panthor_vm_unmap_pages` tears down the pages from the page table
    - it calls `arm_lpae_unmap_pages` from `pgtbl_ops`
      - if a page table has no more mapped pages, it calls `free_pt` to free
        the page
  - `drm_gpuva_unmap` removes `drm_gpuva` from `drm_gpuvm`
  - `drm_gpuva_unlink` removes `drm_gpuva` from `drm_gpuvm_bo`
  - vmas are added to `returned_vmas` and are freed in
    `panthor_vm_cleanup_op_ctx`
- `panthor_vm_active` enables a VM
  - `panthor_mmu_as_enable` sets the regs
  - `transtab` is directly from `cfg->arm_lpae_s1_cfg.ttbr`
    - `virt_to_phys(page_address(vm->root_page_table))`
  - `transcfg` is often `0x420001c6`
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

## GPUVM

- users
  - kernel bo uses `panthor_vm_map_bo_range` to map a bo into a vm and uses
    `panthor_vm_unmap_range` to unmap
  - userspace bo calls `panthor_vm_bind_exec_sync_op` to map/unmap
    synchronously or `panthor_vm_bind_job_create` to map/unmap asynchronously
- high-level view
  - `panthor_vm_prepare_map_op_ctx` and `panthor_vm_prepare_unmap_op_ctx` prep
    a temp `panthor_vm_op_ctx`
  - `panthor_vm_exec_op` calls into gpuvm to map/unmap
    - gpuvm translates the map/unmap to stepwise
      `panthor_gpuva_sm_step_map`/`panthor_gpuva_sm_step_unmap`
  - `panthor_vm_cleanup_op_ctx` cleans up the temp `panthor_vm_op_ctx`
- map
  - `panthor_vm_prepare_map_op_ctx`
    - `panthor_vm_op_ctx_prealloc_vmas` pre-allocs `panthor_vma`s
    - `panthor_gem_pin` pins bo pages
    - `drm_gpuvm_bo_create` and `drm_gpuvm_bo_obtain_prealloc` creates a
      unique `drm_gpuvm_bo` corresponding to the gem bo
    - `panthor_vm_op_ctx_prealloc_pts` pre-allocs pgtables
    - if bo is not exclusive, `drm_gpuvm_bo_extobj_add` adds the bo as extobj
  - `panthor_gpuva_sm_step_map`
    - `panthor_vm_op_ctx_get_vma` and `panthor_vma_init` inits a pre-allocated
      vma
    - `panthor_vm_map_pages` sets up hw pgtable
    - `drm_gpuva_map` adds the vma to the vm (`gpuvm->rb.tree`)
    - `drm_gpuva_link` adds the vma to the vm bo (`vm_bo->list.gpuva`)
    - `drm_gpuvm_bo_put_deferred` derefs the vm bo
  - `panthor_vm_cleanup_op_ctx`
    - `panthor_gem_unpin` unpins the gem bo
    - `drm_gpuvm_bo_deferred_cleanup` cleans up vm bos
- unmap
  - `panthor_vm_prepare_unmap_op_ctx`
  - `panthor_gpuva_sm_step_unmap`
    - `panthor_vm_unmap_pages` tears down hw pgtable
    - `drm_gpuva_unmap` removes the vma from the vm (`gpuvm->rb.tree`)
    - `drm_gpuva_unlink_defer` removes the vma from the vm bo (`vm_bo->list.gpuva`)
- why `drm_gpuvm_bo_deferred_cleanup`?
  - commit 8e4865faf7a97de2a0fd797556a62b31528b42bc
  - vm ops can be on fence signaling path
  - fence signaling path must not sleep
    - e.g., must not take `dma_resv_lock` or `mutex_lock`
  - fence signaling path must alloc with `GFP_NOWAIT`
    - on alloc, shrinker might need to reclaim pages from gem bos on lowmem
    - shrinker might block on fence signaling to have idle gem bos to reclaim
  - we pre-allocate on prep and defers freeing to cleanup

## GEM

- `panthor_kernel_bo_create` creates a kernel bo
  - `drm_gem_shmem_create` creates a shmem gem object
    - this calls `panthor_gem_create_object` to alloc the struct
    - unlike a userspace bo, `drm_gem_handle_create` is not called
  - `panthor_vm_alloc_va` allocs a VA for the bo
    - remember that a VM has a userspace region and a kernel region
    - this allocates the VA from the kernel region, managed by `vm->mm`
  - `panthor_vm_map_bo_range` maps the bo in the vm
    - this uses gpuvm to set up the page tables
  - the kernel bo is exclusive to the vm
- `panthor_kernel_bo_vmap` maps kernel bo for cpu access
- kernel bo users
  - `panthor_fw_load_section_entry` allocs 1 bo per-section for fw
  - `panthor_heap_pool_create` allocs 1 bo per-vm for heap contexts
  - `panthor_alloc_heap_chunk` allocs 1 bo per-chuck for each heap
  - `panthor_fw_alloc_suspend_buf_mem` allocs 2 bos per-group for group
    suspension
  - `panthor_group_create` creates 1 bo per-group for queue seqnos
  - `group_create_queue` creates 2 bos per-queue for ring buffer and fdinfo
    stats respectively
  - `panthor_fw_alloc_queue_iface_mem` allocs 2 pages per-queue for ring
    buffer head/tail
- `panthor_gem_create_with_handle` creates a userspace bo
  - `drm_gem_shmem_create` creates a shmem gem object
  - the bo may be exclusive (private) to the vm, or shareable
  - `drm_gem_handle_create` creates the gem handle for userspace

## FW

- `panthor_fw_init` inits `ptdev->fw`
  - `fw->req_waitqueue` is waitqueue for mcu/glb/csg cmds
  - `fw->sections` is loaded fw sections
  - `fw->watchdog` is watchdog for glb
  - `panthor_request_job_irq` requests no irq sources
    - `panthor_fw_start` will change the irq sources
  - `panthor_gpu_l2_power_on` powers on L2
  - `panthor_vm_create` creates a vm for the fw
  - `panthor_fw_load` loads the fw
  - `panthor_vm_active` enables the vm
  - `panthor_fw_start` boots the fw
  - `panthor_fw_init_ifaces` locates the glb/csg/cs interfaces exported by fw
  - `panthor_fw_init_global_iface` inits the glb interface
- `panthor_job_irq_handler`
  - if `JOB_INT_GLOBAL_IF`, it means mcu has booted or glb req completes
  - if `JOB_INT_CSG_IF(n)`, it means csg N req completes
- `panthor_fw_load`
  - a fw has multiple entries
  - `panthor_fw_load_entry` loads an entry
    - `ehdr` is the entry header
    - `CSF_FW_BINARY_ENTRY_TYPE(ehdr)` returns the entry type
      - `CSF_FW_BINARY_ENTRY_TYPE_IFACE` specifies an interface
      - `CSF_FW_BINARY_ENTRY_TYPE_BUILD_INFO_METADATA` specifies the metadata
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
- `panthor_fw_start`
  - `panthor_job_irq_resume` enables all irq sources
  - it writes `MCU_CONTROL_AUTO` to `MCU_CONTROL`
  - the fw is considered booted after a `JOB_INT_GLOBAL_IF` irq is received
- `panthor_fw_init_ifaces`
  - each interface consists of control, input, and output regions
    - control is for iface props and is read-only 
    - input is written by host and is read-write
    - output is written by mcu and is read-only
  - `panthor_fw_global_iface`
    - there is only one
    - `glb_iface->control` points to `CSF_MCU_SHARED_REGION_START`
    - `glb_iface->{input,output}` are given by
      `glb_iface->control->{input,output}_va`
  - `panthor_fw_csg_iface`
    - there are multiple command stream gropus (e.g., 8) given by
      `glb_iface->control->group_num`
    - `csg_iface->control` are gvien by
      `CSF_GROUP_CONTROL_OFFSET + glb_iface->control->group_stride * idx`
    - `csg_iface->{input,output}` are given by
      `csg_iface->control->{input,output}_va`
  - `panthor_fw_cs_iface`
    - there are multiple command streams per group (e.g., 8) given by
      `csg_iface->control->stream_num`
    - `cs_iface->control` are gvien by
      `CSF_STREAM_CONTROL_OFFSET + csg_iface->control->stream_stride * idx`
      relative to the csg control
      `CSF_GROUP_CONTROL_OFFSET + glb_iface->control->group_stride * idx`
    - `cs_iface->{input,output}` are given by
      `cs_iface->control->{input,output}_va`
    - there are ~128 regs
- communication with mcu
  - host updates the input region
    - `panthor_fw_update_reqs` and `panthor_fw_toggle_reqs` for reqs
    - direct writes for values
  - mcu may read the input region at any time
    - ringing the doorbell ensures mcu reads it timely
    - for reqs,
      - mcu executes `req ^ ack` and sets `ack` to `req`
      - if `req ^ ack` intersects `ack_irq_mask`, an irq is raised
  - host calls `panthor_fw_wait_acks` to wait on mcu
    - it requires the bits to be set in `ack_irq_mask`, to get an irq
  - mcu may also send an event to host proactively
    - mcu sets bits in `ack`, and raises an irq if the bits are in
      `ack_irq_mask`
    - host handles the events and sets `req` to `ack`
- watchdog
  - `panthor_fw_ping_work` is called every `PING_INTERVAL_MS` (12s) to ping
    mcu
- suspend/resume
  - `panthor_fw_suspend` calls `panthor_fw_pre_reset`
    - it sets `GLB_HALT` req, to halt mcu for fast reset
    - it rings the doorbell to notify mcu
    - it waits until `MCU_STATUS` becomes `MCU_STATUS_HALT`
  - `panthor_fw_resume` calls `panthor_fw_post_reset`
    - `panthor_vm_active` enables fw vm
    - if fast reset, it clears `GLB_HALT` req
      - if slow reset, `panthor_reload_fw_sections` reloads fw
    - `panthor_fw_start` starts fw
    - `panthor_fw_init_global_iface` inits gbl

## Heap

- panvk creates a tiler heap for the single `VkQueue`
  - `vm_id` is the device VM
  - `chunk_size` is 2MB
  - `initial_chunk_count` is 5
  - `max_chunks` is 64
  - `target_in_flight` is 65535
- `panthor_vm_get_heap_pool` creates the per-vm heap pool on-demand
  - `pool->gpu_contexts` is the bo for heap contexts
    - each context is `HEAP_CONTEXT_SIZE` (32) bytes, aligned to cacheline
    - there can be `MAX_HEAPS_PER_POOL` (128) contexts
      - this does not appear to be a hw limit, but arbitrarily chosen
- `panthor_heap_create` creates a heap
  - each heap has an id, which is an index into `pool->gpu_contexts`
  - `chunks` is a list of `panthor_heap_chunk`
    - each chunk is a BO
    - each chunk starts with `panthor_heap_chunk_header`, which hw understands
      to form a list of chunks
- when the hw tiler runs out of heap memory, it generates `CS_TILER_OOM` irq
  - `cs_slot_process_irq_locked` schedules `group_tiler_oom_work`
  - `group_process_tiler_oom` grows the heap
  - `cs_iface->output` has info about the tile heap
    - `heap_address` is the addr of the heap context
    - `heap_vt_start`, `heap_vt_end`, and `heap_frag_end` are job seqnos
      - each job goes through vt start, vt end, and frag end
  - `panthor_heap_grow` allocs a new heap chunk
    - it may fail, which is a way to tell the fw to wait
    - it allocs a chunk and returns the va of the new chunk
      - it does not update the header of the previous chunk to link with the
        new chunk
      - i guess the fw is in charge of linking now
  - `cs_iface->input` is used to notify the new heap chunk
    - `heap_start` and `heap_end` are set to the heap chunk

## Scheduler

- FW/HW limits
  - the fw exposes M slots and N queues
  - it means the fw can schedule up to M active groups, with each group having
    N queues
  - when there are less than M active groups in total, the fw can take care of
    scheduling by itself
  - otherwise, the driver must rotate the active groups
- `panthor_sched_init` inits `ptdev->scheduler`
  - `last_tick`, `resched_target`, and `tick_period`
    - `tick_period` is 10ms
    - `last_tick` is updated by `tick_work`
    - `resched_target` is `last_tick + tick_period`, or `U64_MAX` to disable
      ticks
  - `tick_work` decides which groups are active
  - `sync_upd_work`
  - `process_fw_events_work` processes irqs
  - `groups`
    - `runnable`
    - `idle`
    - `waiting`
  - `reset`
    - `stopped_groups`
  - `heap_alloc_wq` is for `group_tiler_oom_work`
  - `wq` is for `drm_gpu_scheduler` and all other works
- `panthor_group_create` creates a group
  - panvk creates a group for the single `VkQueue`
    - all tiler/fragment/compute are allowed
    - priority is determined by global priority
    - vm is the per-device VM
    - there are 3 queues, for tiler, fragment, and compute respectively
  - `state` is `PANTHOR_CS_GROUP_CREATED`
  - `csg_id` is still -1, until it becomes active
  - `vm` points to the specified `panthor_vm`
  - `suspend_buf` and `protm_suspend_buf` are allocated, used by csf to save
    state when an active group is suspended
  - `syncobjs` has one `panthor_syncobj_64b` per queue
    - csf will write seqno to them
    - not to be confused with `drm_syncobj`
  - `group_create_queue` creates a `panthor_queue` for each queue
    - `fence_ctx`
      - `id` is the fence context
      - `in_flight_jobs` is a list of in-flight jobs
    - `ringbuf` is the ring buffer
    - `iface` is allocated, shared between host and csf for ring head/tail
    - `profiling` is allocated, for fdinfo
    - `scheduler` is initialized by `drm_sched_init`
    - `entity` is initialized by `drm_sched_entity_init`
      - each entity has its own 2 fence contexts
      - each job will have two fences, `scheduled` and `finished` from the two
        contexts
  - `idle_queues` is a bitmask of idle queues (all queues initially)
  - the group is added to `sched->groups.idle[group->priority]`
- group and queue lifecycle
  - a group typically has 3 queues, for vert, frag, comp respectively
  - queue lifecycle
    - when the group containing the queue is scheduled, the queue is started
    - when the group containing the queue is evicted, the queue is stopped
    - if the ringbuf is empty, the queue is on `group->idle_queues`
    - if the ringbuf is non-empty, and is blocked by `SYNC_WAIT`, the queue is
      on `group->blocked_queues`
  - group lifecycle
    - when a group is evicted,
      - it is suspended (or terminated if bad) and becomes
        `PANTHOR_CS_GROUP_SUSPENDED` (or `PANTHOR_CS_GROUP_TERMINATED`)
      - it is reset from `csg_slot->group`
      - if all queues are on `group->idle_queues` or `group->block_queues`,
        it is added to `sched->groups.idle`
      - otherwise, it is added to `sched->groups.runnable`
    - when a group is scheduled,
      - it is set to `csg_slot->group`
      - it is started/resumed and becomes `PANTHOR_CS_GROUP_ACTIVE`
      - it is removed from `sched->groups.idle` or `sched->groups.runnable`
- `panthor_job_create` creates a job for each submit
  - `call_info` points to cs instrs
  - `group` is the group
  - `queue_idx` is the queue
  - `done_fence` will be initialized in `queue_run_job` and will signal when
    the job completes
  - `profiling` is for fdinfo
  - `base` is `drm_sched_job`
- when the scheduler calls `queue_run_job` after all deps are met,
  - `panthor_device_resume_and_get` resumes the device
  - `group_can_run` makes sure the group is in a good state
  - `dma_fence_init` inits `job->done_fence` using `queue->fence_ctx`
  - `prepare_job_instrs` preps the call instrs on stack
    - instrs
      - `FLUSH_CACHE2(all)`
      - if profiling
        - `STORE_STATE(cycles)` to `cycles.before`
        - `STORE_STATE(timer)` to `time.before`
      - `WAIT(FLUSH_CACHES2)`
      - `CALL(cmdbuf)`
      - if profiling
        - `STORE_STATE(cycles)` to `cycles.after`
        - `STORE_STATE(timer)` to `time.after`
      - `WAIT(all)`
      - `SYNC_ADD64(&group->syncobjs[queue], 1)`
        - this has system scope and will trigger csg irq with `CSG_SYNC_UPDATE`
    - the instrs use the top `unpreserved_cs_reg_count` regs
      - there are `CSF_UNPRESERVED_REG_COUNT` (4) regs
      - it is exposed to userspace and userspace knows that a submit will
        trash the top `unpreserved_cs_reg_count` regs
  - `copy_instrs_to_ringbuf` copies the call instrs to ringbuf
  - `job->ringbuf` saves the start/end positions the the call instrs in the
    ringbuf
  - the job is added to `queue->fence_ctx.in_flight_jobs`
  - the ringbuf tail is updated
  - if the group is inactive, `group_schedule_locked` queues `tick_work`
  - if the group is active, it rings cs doorbell
    - `panthor_devfreq_record_busy` reports busy to devfreq
  - `queue->fence_ctx.last_fence` is set to `job->done_fence`
- when the scheduler calls `queue_timedout_job`
  - `drm_sched_stop` stops the scheduler for the queue
  - `group->timedout` is set
  - it queues `tick_work` or `group_term_work`, depending on if the group is
    active
  - `drm_sched_start` starts the scheduler again
- on job irq, `panthor_request_job_irq` calls `panthor_sched_report_fw_events`
  to queue `process_fw_events_work`
  - `sched_process_global_irq_locked` processes glb irq
    - `sched_process_idle_event_locked` processes `GLB_IDLE`
      - it queues `tick_work`
  - `sched_process_csg_irq_locked` processes csg irq
    - it updates `req` and `cs_irq_ack` to ack csg events and cs irqs
    - `csg_slot_process_idle_event_locked` processes `CSG_IDLE`
      - it marks `might_have_idle_groups` and queues `tick_work`
    - `csg_slot_process_progress_timer_event_locked` processes `CSG_PROGRESS_TIMER_EVENT`
      - it marks `group->timedout` and queues `tick_work`
    - `cs_slot_process_irq_locked` processes cs irq
      - `cs_slot_process_fatal_event_locked` processes `CS_FATAL`
        - it marks `group->fatal_queues`, queues `tick_work` or
          `panthor_device_reset_work` depending on the error, and logs the
          error
      - `cs_slot_process_fault_event_locked` processes `CS_FAULT`
        - if `DRM_PANTHOR_EXCEPTION_CS_INHERIT_FAULT`, it marks the offending
          job error and logs the error
      - `cs_slot_process_tiler_oom_event_locked` processes `CS_TILER_OOM`
        - it marks `group->tiler_oom` and queues `group_tiler_oom_work`
    - `csg_slot_sync_update_locked` processes `CSG_SYNC_UPDATE`
      - it queues `group_sync_upd_work` and `sync_upd_work`
- `panthor_csg_slots_upd_ctx`
  - `csgs_upd_ctx_queue_reqs` batches csg reqs
  - `csgs_upd_ctx_apply_locked` commits csg reqs
    - `panthor_fw_update_reqs` updates `csg_iface->input->req`
    - `panthor_fw_ring_csg_doorbells` rings csg doorbells
    - `panthor_fw_csg_wait_acks` waits for mcu
  - `csg_slot_sync_priority_locked` processes `CSG_ENDPOINT_CONFIG`
    - it updates `csg_slot->priority`, after the mcu has acked the
      user-specified priority
  - `csg_slot_sync_state_locked` processes `CSG_STATE_MASK`
    - it updates `group->state`, after the mcu has changed the csg state
    - depending on the old/new states, they might trigger
      - `cs_slot_reset_locked`, which toggles `CS_STATE_STOP`
      - `csg_slot_sync_queues_state_locked`, when `CSG_STATUS_UPDATE` is
        implied
      - `panthor_device_schedule_reset`, when things go bad
  - `csg_slot_sync_queues_state_locked` processes `CSG_STATUS_UPDATE`
    - it updates `group->idle_queues`, `group->blocked_queues`,
      `queue->syncwait`, and more
  - `csg_slot_sync_idle_state_locked` processes `CSG_STATUS_UPDATE`
    - it updates `csg_slot->idle`, based on `CSG_STATUS_STATE_IS_IDLE`
- `tick_work` schedules groups
  - it runs periodically (`tick_period`) when there are more groups than the
    fw can handle, to rotate them
    - otherwise, it runs after triggered by external events (e.g., submit to
      an inactive group, irqs)
  - `tick_ctx_init` inits `panthor_sched_tick_ctx`
    - `tick_ctx_insert_old_group` adds active groups to `ctx->old_groups`
      - `full_tick` is true if the tick is caused by external events
    - it toggles `CSG_STATUS_UPDATE` for active groups
  - `tick_ctx_pick_groups_from_list` picks groups and moves them to
    `ctx->groups`
  - `tick_ctx_apply`
    - for `ctx->old_gorups`, it toggles `CSG_STATE_SUSPEND` to suspend them
      - if they are in a bad state, it toggles `CSG_STATE_TERMINATE` to
        terminate them
    - for `ctx->groups`, if already running, it toggles `CSG_ENDPOINT_CONFIG`
      to change the priority
    - for `ctx->old_gorups`,
      - `sched_process_csg_irq_locked` processes (pending) csg irq
      - `group_unbind_locked` unbinds the group from the csg slot
    - for `ctx->gorups`,
      - `group_bind_locked` binds the group to the csg slot
      - `csg_slot_prog_locked`
        - for each queue, `cs_slot_prog_locked` inits the input section
          - the ring buffer is set up
          - `ack_irq_mask` is enabled for all sources
          - `req` has
            - `CS_IDLE_SYNC_WAIT` to treat `SYNC_WAIT` as idle
            - `CS_IDLE_EMPTY` to treat ring buffer empty as idle
            - `CS_STATE_START` to start the cs
            - `CS_EXTRACT_EVENT` is not used?
        - for csg itself, it inits the input section
          - `ack_irq_mask` is enabled for all sources
        - it rings the cs doorbells
      - it toggles `CSG_STATE_START` or `CSG_STATE_RESUME` to start/resume the
        csg
      - it toggles `CSG_ENDPOINT_CONFIG`
  - `panthor_devfreq_record_idle` or `panthor_devfreq_record_busy` is called,
    depending on whether any group is busy
  - it reschedules the next tick
- when a job completes, an irq is generated
  - this is because `prepare_job_instrs` emits `SYNC_ADD64.system_scope`,
    which sets `CSG_SYNC_UPDATE`
  - `csg_slot_sync_update_locked` handles the event and queues two works in
    order
  - `group_sync_upd_work`
    - it signals `job->done_fence` by comparing seqnos
    - `update_fdinfo_stats` accumulates profiling data
  - `sync_upd_work`
    - `panthor_queue_eval_syncwait` decides if the queue has been unblocked
- suspend/resume
  - `panthor_sched_suspend`
    - it suspends or terminates all csgs
    - `panthor_gpu_flush_caches` flushes caches
    - it unbinds all running groups and adds them to `sched->groups.idle`
  - `panthor_sched_resume`
    - it queues `tick_work`
- reset
  - `panthor_sched_pre_reset`
    - it cancels `sync_upd_work` and `tick_work`
    - `panthor_sched_suspend` suspends all running groups
    - for idle or runnable groups, `panthor_group_stop` calls `drm_sched_stop`
      on their queues and moves them to `sched->reset.stopped_groups`
  - `panthor_sched_post_reset`
    - for groups moved to `sched->reset.stopped_groups`, `panthor_group_start`
      calls `drm_sched_start` on their queues and moves them back to
      `sched->groups.idle` or `sched->groups.runnable`
    - it queues `tick_work` and `sync_upd_work`
- priority
  - groups and queues both have userspace-specified priorities
    - `panthor_group_create` inits `group->priority` to one of
      - `PANTHOR_CSG_PRIORITY_LOW`
      - `PANTHOR_CSG_PRIORITY_MEDIUM`
      - `PANTHOR_CSG_PRIORITY_HIGH`
      - `PANTHOR_CSG_PRIORITY_RT`
      - this is vk global priority
    - `group_create_queue` inits `queue->priority` between `[0, CSF_MAX_QUEUE_PRIO]`
      - panvk always uses 1
  - queue priority is used by hw directly with `CS_CONFIG_PRIORITY`
  - group priority is used by hw indirectly
    - `tick_ctx_apply` assigns each group a different hw prio starting from
      `MAX_CSG_PRIO`
    - hw prio is used with `CSG_EP_REQ_PRIORITY`
- group tracking with `group->run_node`
  - `panthor_group_create` adds a group to
    `sched->groups.idle[group->priority]` initially
  - when a job is submitted, `group_schedule_locked` moves the group to
    `sched->groups.runnable[group->priority]`
  - upon tick,
    - `tick_work` moves the group to `ctx->groups[group->priority]`
    - `tick_ctx_apply` removes the group from any list
      - instead, `csg_slot->group` points the group now
  - when userspace destroys the group, it triggers another tick
    - `tick_ctx_insert_old_group` adds the active group to
      `ctx->old_groups[group->priority]`
    - `tick_ctx_apply` suspends the active group
    - `tick_ctx_cleanup` removes the group from any list and schedules
      `group_term_work`
- `tick_work` group rotation
  - `tick_ctx_init` calls `tick_ctx_insert_old_group` on each active group
    - it adds each active group to `ctx->old_groups[group->priority]`
    - it also sort active groups by hw prios
      - almost all groups are `PANTHOR_CSG_PRIORITY_MEDIUM` and are on the same list
      - groups on the same list are sorted by their hw prios
  - it loops to pick from active and runnable groups
    - `tick_ctx_pick_groups_from_list` moves active groups from
      `ctx.old_groups[prio]` to `ctx->groups[group->priority]`
    - `tick_ctx_pick_groups_from_list` moves runnable groups from
      `sched->groups.runnable[prio]` to `ctx->groups[group->priority]`
    - note that the active group with the highest hw prio is moved to the tail
      - this is to assign it a lower hw prio and assign higher hw prios to
        other groups with the same group priority
      - remember that almost all groups are `PANTHOR_CSG_PRIORITY_MEDIUM` and
        we don't want one group to starve others with the same group priority
  - it loops again to pick from idle groups
    - same as above, but this time idle groups are considered
  - `tick_ctx_apply`
    - if an active group is not picked and is still on
      `ctx->old_groups[prio]`, it is suspended
    - if an active group is picked, it is assigned a new hw prio
    - if a runnable/idle group is picked, it is started
    - active groups are no longer tracked on any list
      - instead, they are pointed to by `csg_slot->group`
    - suspended groups are moved from `ctx->old_groups[prio]` to
      `sched->groups.idle[prio]` or `sched->groups.runnable[prio]` if they
      still can run
  - `tick_ctx_cleanup`
    - suspended groups that can no longer run are sent to `group_term_work`

## Error Recovery

- a CS is either enabled or disabled
  - the CS is initially disabled and does not fetch nor execute instrs
  - `CS_STATE_START` req enables the CS to start fetching and executing instrs
  - upon `CS_FAULT` or `CS_FATAL`, the CS is still enabled but stops fetching
    and executing instrs; the possible operations are
    - `CS_STATE_STOP` req to disable just the CS
    - `CSG_STATE_TERMINATE` req to terminate the entire CSG
    - if `CS_FAULT`, a third option is to ack `CS_FAULT` to enter error recovery
      - the CS will fetch and execute instrs again, but the error state will
        be set
      - `RUN_*` will be nop
      - `SYNC_*` with `error_propagate` will propagate 1
      - `ERROR_BARRIER` will clear the error state and return the CS to normal
        enabled state
  - `CS_STATE_STOP` req disables the CS
- a CSG is either enabled or disabled
  - the CSG is initially disabled; all CS reqs are ignored
  - `CSG_STATE_START` req enables the CSG; only `CS_STATE_START` req is
    honored; after the CS is enabled, all CS reqs are honored
  - `CSG_STATE_TERMINATE` req disables the CSG; it disables all CSs as well
  - alternatively, `CSG_STATE_SUSPEND` and `CSG_STATE_RESUME` reqs also
    disable/enable the CSG
    - the difference is that they also save/restore CS states
- on mmu fault, an mmu irq is raised
  - `panthor_mmu_irq_handler` disables the AS, marks `unhandled_fault`, and
    logs the fault
  - `panthor_sched_report_mmu_fault` queues `tick_work`
- on gpu fault, a gpu irq is raised
  - `panthor_gpu_irq_handler` logs the fault
- on cs fault or job stall, a job irq is raised
  - `panthor_job_irq_handler` queues `process_fw_events_work`
  - `process_fw_events_work` calls `sched_process_csg_irq_locked` on csgs that
    need attention, which calls `cs_slot_process_irq_locked` on CSs that need
    attention
  - `csg_slot_process_progress_timer_event_locked` handles
    `CSG_PROGRESS_TIMER_EVENT`
    - it logs an error, marks `group->timedout`, and queues `tick_work`
  - `cs_slot_process_fatal_event_locked` handles `CS_FATAL`
    - it marks `group->fatal_queues`, queues `tick_work` other than
      `DRM_PANTHOR_EXCEPTION_CS_UNRECOVERABLE`, and logs an error
  - `cs_slot_process_fault_event_locked` handles `CS_FAULT`
    - it marks the fence error for `DRM_PANTHOR_EXCEPTION_CS_INHERIT_FAULT`,
      and logs an error
  - `cs_slot_process_tiler_oom_event_locked` handles `CS_TILER_OOM`
    - it grows tiler heap and is normally fine
    - on system oom, it logs an error, marks `group->fatal_queues`, and queues
      `tick_work`
- on scheduler timeout, `queue_timedout_job` is called
  - it logs an error, marks `group->timedout`, and queues `tick_work`
- on `tick_work`,
  - if `panthor_vm_has_unhandled_faults(group->vm)`, it marks
    `group->fatal_queues`
  - if `!group_can_run(group)`, implied by `group->timedout` or
    `group->fatal_queues`, it queues `group_term_work`
    - `tick_ctx_insert_old_group` moves them to `ctx->old_groups`
    - `tick_ctx_pick_groups_from_list` never moves them to `ctx->groups`
    - `tick_ctx_apply` sends `CSG_STATE_TERMINATE` reqs to `ctx->old_groups`
    - `tick_ctx_cleanup` queues `group_term_work` for them
- on `group_term_work`,
  - it signals all fences
- `panthor_device_schedule_reset` is called from several places
  - from `wait_ready` when mmu `AS_COMMAND` times out
  - from `panthor_fw_ping_work` when mcu `GLB_PING` times out
  - from `csgs_upd_ctx_apply_locked` error handling, when any CSG req times
    out
  - from `csg_slot_sync_state_locked` when `tick_work` sends `CSG_STATE_MASK`
    req and gets unexpected states
  - from `cs_slot_process_fatal_event_locked` when `CS_FATAL` is
    `DRM_PANTHOR_EXCEPTION_CS_UNRECOVERABLE`

## Experiments with Error Recovery

- when a shader has a infinite loop, it hits `PROGRESS_TIMEOUT_CYCLES`,
  `JOB_TIMEOUT_MS`, or both
  - if `PROGRESS_TIMEOUT_CYCLES`, we get a job irq
    - `process_fw_events_work` is the bottom half
    - `csg_slot_process_progress_timer_event_locked` handles
      `CSG_PROGRESS_TIMER_EVENT`
      - it prints `CSG slot %d progress timeout`
      - it marks `group->timedout`
      - it queues `tick_work`
    - `tick_work` terminates the group
      - because `group_can_run` will return false for the group, the group is
        on `ctx->old_groups`
      - `tick_ctx_apply` will `CSG_STATE_TERMINATE` the group
      - `tick_ctx_cleanup` will queue `group_term_work`
    - `group_term_work` post-processes the terminated group
      - all fences are signaled with `-ETIMEDOUT`
    - umd reports device lost
      - if it `DRM_IOCTL_PANTHOR_GROUP_GET_STATE`, it sees `group->timedout`
      - if it `DRM_IOCTL_PANTHOR_GROUP_SUBMIT`, it gets rejected
  - if `JOB_TIMEOUT_MS`, `queue_timedout_job` is called
    - it prints `job timeout`
    - it marks `group->timedout`
    - it queues `tick_work`
    - the rest is the same
- when a bo is freed prematurely,
  - we get an mmu irq, handled by `panthor_mmu_irq_handler`
    - it prints `Unhandled Page fault in AS%d at VA 0x%016llX`
    - it mask out the AS for further irqs
    - it marks `vm->unhandled_fault`
    - `panthor_mmu_as_disable` disables the vm
      - cs appears blocked by mmu for the memory access
      - by disabling the vm, the fault is propgated and cs gets a fatal
        `CS_BUS_FAULT`
      - otherwise, we have to wait until cs times out
    - `panthor_sched_report_mmu_fault` queues `tick_work`
  - we also get a job irq, handled by `panthor_job_irq_handler`
    - this only because `panthor_mmu_as_disable` disables mmu
      - the offending cs gets fatal `CS_BUS_FAULT`
      - the remaining cs gets faulty `CS_RES_TERM`
      - some cs get both
    - `process_fw_events_work` is the bottom half
    - `sched_process_csg_irq_locked` handles the offending csg
    - `cs_slot_process_irq_locked` handles each cs
    - `cs_slot_process_fatal_event_locked` handles `CS_FATAL`
      - it marks `group->fatal_queues`
      - it queues `tick_work` or `panthor_device_reset_work` depending on the
        exception type
      - it prints `CSG slot %d CS slot: %d ... CS_FATAL...`
    - `cs_slot_process_fault_event_locked` handles `CS_FAULT`
      - it marks some fences error on `CS_INHERIT_FAULT`
        - `CS_INHERIT_FAULT` means the cs is executing `SYNC_WAIT` and
          receives a propagated error
        - marking a fence error does not signal the fence, and is mostly
          useless?
      - it prints `CSG slot %d CS slot: %d ... CS_FAULT...`
  - `tick_work`
    - it checks `panthor_vm_has_unhandled_faults` and calls
      `sched_process_csg_irq_locked` too
      - if `tick_work` races with `process_fw_events_work`, it makes sure
        `group->fatal_queues` is set for this tick
    - `group_can_run` returns false if `group->fatal_queues`
    - similarly to job timeout, `tick_ctx_apply` terminates the group and
      `group_term_work` signals all fences with `-EINVAL` or `-ECANCELED`
- devcoredump ideas
  - on mmu fault, `panthor_mmu_irq_handler` can, instead of marking
    `vm->unhandled_fault`, collect per-vm `AS_FAULTSTATUS` and
    `AS_FAULTADDRESS`
  - on gpu fault, `panthor_gpu_irq_handler` can collect `GPU_FAULT_STATUS` and
    `GPU_FAULT_ADDR`
    - it should be cleared with `GPU_CLEAR_FAULT` cmd
  - on cs fault, `panthor_job_irq_handler` can collect per-CS `CS_FAULT`,
    `CS_FATAL`, `CS_FAULT_INFO`, `CS_FATAL_INFO`, `CS_EXTRACT`, etc.
  - general gpu regs
    - `GPU_ID`
    - `GPU_STATUS`
    - `SHADER_READY`
    - `TILER_READY`
    - `L2_READY`
    - `MCU_STATUS`
  - general mmu regs
    - `STATUS`

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
  - `panthor_vm_get_heap_pool` returns (or creates on demand) the single heap
    pool for the VM
    - `pool->gpu_contexts` is a kernel bo
  - `panthor_heap_create` creates a heap
    - each chunk is a kernel bo
  - I guess this allocates buffers for use by the fw
- `DRM_IOCTL_PANTHOR_TILER_HEAP_DESTROY` maps to `panthor_ioctl_tiler_heap_destroy`
- `DRM_IOCTL_PANTHOR_GROUP_CREATE` maps to `panthor_ioctl_group_create`
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

## `panthor_ioctl_vm_create`

- panvk creates a VM for a `VkDevice`
  - `user_va_range` is 4GB
  - panvk manages the address space and uses `[32MB, 4GB)`
- `panthor_vm_create_check_args`
  - `full_va_range` is 4TB (`va_bits` is 48)
  - `user_va_range` is 4GB (determined by panvk)
  - `kernel_va_range` is thus the top 2TB
- each `panthor_file` has a `panthor_vm_pool` to manage its vms

## `panthor_ioctl_bo_create` and `panthor_ioctl_vm_bind`

- when panvk allocates a `VkDeviceMemory`, it allocs bo and binds vm
  immediately atm
  - `DRM_IOCTL_PANTHOR_BO_CREATE`
    - `size` is the mem alloc size
    - `exclusive_vm_id` is set when the mem is not external
  - `DRM_IOCTL_PANTHOR_VM_BIND`
    - `flags` is 0 (synchronous)
    - `ops`
      - `flags` is `DRM_PANTHOR_VM_BIND_OP_TYPE_MAP` or
        `DRM_PANTHOR_VM_BIND_OP_TYPE_UNMAP`
      - `va` and `size` are managed by panvk
- `panthor_ioctl_bo_create`
  - `drm_gem_shmem_create` creates `panthor_gem_object`
    - `panthor_gem_create_object` allocs the obj struct
    - `drm_gem_object_init_with_mnt` inits the obj struct and calls
      `shmem_file_setup` to alloc a shmem
    - `drm_gem_create_mmap_offset` creates a magic mmap offset for mmap
  - if exclusive, `bo->resv` uses the vm's `resv`
  - `drm_gem_handle_create` creates a gem handle
- `panthor_ioctl_vm_bind`
  - panvk only does sync bind atm
  - `panthor_vm_pool_get_vm` looks up the vm
  - `panthor_vm_bind_exec_sync_op` executes an op synchronously
    - `panthor_vm_bind_prepare_op_ctx`
      - for mapping, `panthor_vm_prepare_map_op_ctx`
        - `panthor_vm_op_ctx_prealloc_vmas` pre-allocs vma structs
        - `drm_gem_shmem_get_pages_sgt` pins pages and returns sgt
        - `drm_gpuvm_bo_create` and `drm_gpuvm_bo_obtain_prealloc` return a
          gpuvm bo
        - `kmem_cache_alloc_bulk` pre-allocs page tables
        - `drm_gpuvm_bo_extobj_add` adds the gpuvm bo to the gpuvm's extobj
          list, if the bo is external
      - for unmapping, `panthor_vm_prepare_unmap_op_ctx`
    - `panthor_vm_exec_op` calls `drm_gpuvm_sm_map` or `drm_gpuvm_sm_unmap`
      - gpuvm calls `panthor_gpuva_sm_step_map` for mapping
        - `panthor_vma_init` inits a new `panthor_vma`
        - `panthor_vm_map_pages` calls into `vm->pgtbl_ops` to set up page
          tables and calls `panthor_vm_flush_range` to flush the mmu
        - `drm_gpuva_map` adds the vma to gpuvm
        - `panthor_vma_link` adds the vma to the gpuvm bo
      - gpuvm calls `panthor_gpuva_sm_step_unmap` for unmapping
        - `panthor_vm_unmap_pages` calls into `vm->pgtbl_ops` to take down the
          page tables and calls `panthor_vm_flush_range` to flush the mmu
        - `drm_gpuva_unmap` removes the vma from gpuvm
        - `panthor_vma_unlink` removes the vma from gpuvm bo
    - `panthor_vm_cleanup_op_ctx`

## `panthor_ioctl_group_submit`

- for a gfx pipeline with waits and signals, panvk typically submits 6 jobs
  - there are 2 empty submits for semaphore waits
    - only `syncs` is set, which is an `drm_panthor_sync_op` array for wait
      semaphores
      - `flags` is `DRM_PANTHOR_SYNC_OP_WAIT` plus
        - `DRM_PANTHOR_SYNC_OP_HANDLE_TYPE_TIMELINE_SYNCOBJ` or
          `DRM_PANTHOR_SYNC_OP_HANDLE_TYPE_SYNCOBJ`
      - `handle` is the semaphore
      - `timeline_value` is the timeline value or 0
    - there are 2 submits against tiler and fs subqueues respectively
  - there are 2 cmd submits
    - `stream_size` is the cmd size
    - `stream_addr` is the cmd v
    - `latest_flush` is the flush id
    - there are 2 submits against tiler and fs subqueues respectively
  - there are 2 empty submits for semaphore and fence signals
    - only `syncs` is set, which is a single `drm_panthor_sync_op`
      - panvk always signals the same per-queue syncobj
      - `flags` is `DRM_PANTHOR_SYNC_OP_SIGNAL` plus
        `DRM_PANTHOR_SYNC_OP_HANDLE_TYPE_TIMELINE_SYNCOBJ`
      - `handle` is the per-queue syncobj
      - `timeline_value` depends on the subqueues
    - there are 2 submits against tiler and fs subqueues respectively
      - `timeline_value` is 1 and 2 respectively
  - panvk post-processes semaphore and fence signals
    - `drm_syncobj_transfer_ioctl` copies the dma-fence from the per-queue
      syncobj to each semaphores and fence signaled
    - `drm_syncobj_reset_ioctl` resets the per-queue syncobj
- `panthor_submit_ctx_init` inits `panthor_submit_ctx`
  - `ctx->jobs` is an array of `panthor_job_ctx`
  - `ctx->exec` is a `drm_exec`
- `panthor_job_create` creates a `panthor_job` for each job
  - it is a sublass of `drm_sched_job`
  - `drm_sched_job_init` inits the `drm_sched_job`
  - `job->done_fence` is allocated only if the job is non-empty
- `panthor_submit_ctx_add_job` adds the job to `ctx->jobs` together with its
  syncops
- `panthor_submit_ctx_collect_jobs_signal_ops` loops through `ctx->jobs` to
  collect signal ops
  - `panthor_check_sync_op` makes sure a binary syncobj always has 0 as the
    timeline value
  - `panthor_submit_ctx_get_sync_signal` makes sure each `(syncobj, point)`
    pair has a `panthor_sync_signal`
  - `panthor_submit_ctx_add_sync_signal` allocs a `panthor_sync_signal` for
    each `(syncobj, point)` pair and adds it to `ctx->signals`
    - if timeline, `sig_sync->chain` points to a new `dma_fence_chain`
    - if `(syncobj, point)` already has a dma-fence, `sig_sync->fence` points
      to the fence
- `panthor_vm_prepare_mapped_bos_resvs`
  - `drm_gpuvm_prepare_vm` preps the vm
  - `drm_gpuvm_prepare_objects` preps the bos
- `panthor_submit_ctx_add_deps_and_arm_jobs` loops through `ctx->jobs`
  - `panthor_submit_ctx_add_sync_deps_to_job` finds the dma-fence for each
    waited `(syncobj, point)` and calls `drm_sched_job_add_dependency` to add
    the fence as a dependency
  - `drm_sched_job_arm` arms a `drm_sched_job`
    - specifically, it inits `job->s_fence`
  - `panthor_submit_ctx_update_job_sync_signal_fences` updates
    `sig_sync->fence` to point to `job->s_fence->finished` from the scheduler
- `panthor_submit_ctx_push_jobs` submits jobs
  - `drm_sched_entity_push_job` pushes each job to the scheduler
  - `panthor_submit_ctx_push_fences` updates signaled syncobjs
    - if timeline, `drm_syncobj_add_point` adds a point
    - if binary, `drm_syncobj_replace_fence` replaces the fence
- when all deps are met, `queue_run_job` runs a job
  - if the job is empty, `job->done_fence` is set to
    `queue->fence_ctx.last_fence` and the function early returns
  - otherwise, `job->done_fence` is initialized and
    `queue->fence_ctx.last_fence` is set to it
  - depending on if the group is active, it either rings a doorbell or calls
    `group_schedule_locked` to schedule
    - `tick_work` will rotate groups if there are more groups than the fw can
      handle
- when a job completes, `panthor_job_irq_handler` handles the irq
  - `CSG_SYNC_UPDATE` indicates job completion
  - `group_sync_upd_work` checks the seqnos and calls
    `dma_fence_signal_locked` on `job->done_fence`
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
