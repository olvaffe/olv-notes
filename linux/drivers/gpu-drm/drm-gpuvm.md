# DRM GPUVM

## History

- 2002.03: i915 proposed `VM_BIND` but never merged
  - it allows userspace to manage GPU VMs
  - no bo list is needed on submission
- 2023.01: nouveau proposed `VM_BIND`
  - a generic DRM GPUVA manager is also introduced
- 2023.08: nouveau gained `VM_BIND` support
- 2023.09: GPUVA growed into GPUVM
  - gem bos exclusive to a vm can share the same dma-resv and be locked
    altogether
- 2023.12: xe merged with `VM_BIND`
- 2024.03: panthor merged with `VM_BIND`
- 2025.07: msm gained `VM_BIND` support

## Typical Driver Flows

- exec job or vm job submission
  - assume an exec op or vm op that waits N syncobjs and signals M syncobjs
  - create a job for the op
    - `drm_sched_job_init` inits the job
    - if vm op,
      - if vm map op,
        - `drm_gpuvm_bo_obtain_locked` preallocs unique per-vm vm bo
          (`drm_gpuvm_bo`)
        - prealloc pgtables
        - prealloc obj pages and sgt
        - if extobj, `drm_gpuvm_bo_extobj_add`
      - prealloc vmas (`drm_gpuva`)
    - `drm_sched_job_add_syncobj_dependency` adds each of the N syncobj to wait
      as deps
    - collect M syncobjs to signal
      - `drm_syncobj_find` looks up the syncobj
      - if timeline, `dma_fence_chain_alloc` allocs a fence chain
  - lock gem objs
    - `drm_exec_init` with `DRM_EXEC_IGNORE_DUPLICATES`
    - `drm_exec_until_all_locked` is the retry loop
      - `drm_gpuvm_prepare_vm` locks `vm->r_obj->resv`
      - if exec op, `drm_gpuvm_prepare_objects` locks extobjs
      - if vm map op, `drm_exec_prepare_obj` locks the obj to be mapped
      - `drm_exec_retry_on_contention` retries on contention
    - if exec op, `drm_gpuvm_validate` validates (swaps in) evicted objs
      - no pinning needed with gpuvm
      - instead, driver validates evicted objs
        - swaps in pages
        - restores gpu pgtable
        - `drm_gpuvm_bo_evict(vm_bo, false)` marks non-evicted
  - queue the job atomically
    - `drm_sched_job_arm` arms the job and inits the out-fence
      - `job->s_fence->finished` is the out-fence
    - point of no return
      - the out-fence must signal in finite time
      - all preps that can fail should happen before this
    - in fact, it can still fail until the out-fence is used
      - if the out-fence has no user, it is harmless to not push the job and
        signal the out-fence
    - `drm_gpuvm_resv_add_fence` adds the out-fence with
      `DMA_RESV_USAGE_BOOKKEEP`
      - this happens before `drm_sched_entity_push_job` to avoid a window where
        an obj is accessed by gpu without an associated fence
    - `drm_sched_entity_push_job` pushes the job
      - this transfer the job to the scheduler
      - the job will run after all deps are satisfied
  - signal M syncobjs
    - if timeline, `drm_syncobj_add_point`
    - if binary, `drm_syncobj_replace_fence`
  - unlock gem objs
    - `drm_exec_fini` unlocks all
- vm job execution
  - if vm map op, `drm_gpuvm_sm_map` calls step map/unmap/remap
  - if vm unmap op, `drm_gpuvm_sm_unmap` calls step unmap/remap
  - step map
    - set up pgtables
    - grab a preallocated vma
    - `drm_gpuva_map` adds vma to vm
    - `drm_gpuva_link` adds vma to vm bo
  - step unmap
    - tear down pgtables
    - `drm_gpuva_unmap` removes vma from vm
    - `drm_gpuva_unlink_defer` or `drm_gpuva_unlink` removes vma from vm bo
      - if vm job execution is on fence signaling path, it must not destroy
        obj because some drivers can hold `obj->resv` on their obj destroy
      - but `drm_gpuva_unlink` might destroy the vm bo, which in turn might
        destroy the obj; `drm_gpuva_unlink_defer` to the rescue
    - free vma
  - step remap consists of a step unmap and one or two step map
- vm job cleanup
  - if vm map op, `drm_gpuvm_bo_put_deferred` derefs the preallocated unique
    per-vm vm bo
  - `drm_gpuvm_bo_deferred_cleanup` destroys defer-destroyed vm bos
    - this must not be on the fence signaling path
- shrinker reclaim
  - `drm_gem_for_each_gpuvm_bo` loops through each vm bo
    - `drm_gpuvm_prepare_vm` locks each vm
    - `drm_gpuvm_bo_evict(vm_bo, true)` marks evicted
    - `drm_gpuvm_bo_for_each_va` loops through each vma
      - tear down pgtables

## Basics

- `drm_gpuvm` is the gpu va manager
- `drm_gpuvm_bo` is unique between each `drm_gpuvm` and `drm_gem_object`
  combo
  - that is, if a gem object is mapped in a vm, there is a unique
    `drm_gpuvm_bo` to track the mapping
  - `drm_gpuvm_bo_obtain` guarantees the uniqueness
    - there is a `obj->gpuva` list to track all `drm_gpuvm_bo`
    - `drm_gpuvm_bo_find` searches the list for a match
    - if found, it returns the unique `drm_gpuvm_bo`
    - otherwise, it creates a new `drm_gpuvm_bo` and adds it to the obj
- `drm_gpuvm_bo_extobj_add` adds a `drm_gpuvm_bo` to a `drm_gpuvm`
  - when a bo is exclusively mapped by a single vm, driver can override
    `obj->resv` to `drm_gpuvm_resv(vm)`
  - `drm_gpuvm_is_extobj` returns false and the bo is not tracked
  - otherwise, the bo is considered an external object and is added to
    `gpuvm->extoj`
- when used with `drm_exec`,
  - `drm_gpuvm_prepare_vm` calls `drm_exec_prepare_obj` on the dummy
    `gpuvm->r_obj`
  - `drm_gpuvm_prepare_objects` calls `drm_exec_prepare_obj` on all external
    objects
- when `drm_gpuvm_sm_map` maps with split & merge,
  - it loops over all existing `drm_gpuva` overlapping with the newly
    requested mapping
    - if the existing gpuva is a subset of the new mapping, the existing gpuva
      is `sm_step_unmap`ed
    - otherwise, it is `sm_step_remap`ed
  - finally, the newly requested mapping is `sm_step_map`ed
    - driver is expected to update the paging table, and calls `drm_gpuva_map`
      and `drm_gpuva_link`
    - `drm_gpuva_map`
      - `drm_gpuva_init_from_op` inits `drm_gpuva`
      - `drm_gpuva_insert` adds the `drm_gpuva` to `gpuvm->rb.tree`
    - `drm_gpuva_link` adds the `drm_gpuva` to `vm_bo->list.gpuva`

## Locking

- `drm_gem_object`
  - `refcount` is refcount
  - `resv` can point to local `_resv` or `vm->r_obj->resv`
    - if an obj is exclusive to a vm, set `obj->resv` to `vm->r_obj->resv`
  - `gpuva.list` is the list of per-vm `drm_gpuvm_bo`
    - it ensure each obj has a unique vm bo in each vm
    - it is protected by `resv` by default
  - `gpuva.lock` is unsed by default
- `drm_gpuvm`
  - `kref` is refcount
  - `rb.tree` is the rb tree of `drm_gpuva`
    - it is protected by a driver mutex
  - `rb.list` is the list of `drm_gpuva` sorted by start va
    - it is protected by a driver mutex
  - `r_obj` is the dummy obj for `obj->resv`
  - `extobj.list` is the list of external `drm_gpuvm_bo`
    - that is, `vm_bo->obj->resv != vm->r_obj->resv`
    - it is protected by `extobj.lock` by default
  - `extobj.lock` protectes `extobj.list` by default
  - `evict.list` is the list of evicted `drm_gpuvm_bo`
    - it is protected by `evict.lock` by default
  - `evict.lock` protectes `evict.list` by default
  - `bo_defer` is the list of defer-destroyed `drm_gpuvm_bo`
    - it requires `DRM_GPUVM_IMMEDIATE_MODE`
- `drm_gpuvm_bo`
  - `kref` is refcount
  - `list.gpuva` is the list of `drm_gpuva`
    - it is protected by `obj->resv` by default
  - `entry.gem` is for `obj->gpuva.list`
  - `entry.extobj` is for `vm->extobj.list`
  - `entry.evict` is for `vm->evict.list`
  - `entry.bo_defer` is for `vm->bo_defer`
- `drm_gpuva`
  - no refcount
  - `rb.node` is for `vm->rb.tree`
  - `rb.entry` is for `vm->rb.list`
  - `gem.entry` is for `vm_bo->list.gpuva`
- if `DRM_GPUVM_RESV_PROTECTED`, `vm->extobj.lock` and `vm->evict.lock` become
  unused
  - driver must lock `vm->r_obj->resv` externally to protect `vm->extobj.list`
    and `vm->evict.list`
  - see `drm_gpuvm_resv_protected` and `drm_gpuvm_resv_assert_held`
- if `DRM_GPUVM_IMMEDIATE_MODE`, `obj->gpuva.lock` replaces `obj->resv` to
  protect `obj->gpuva.list` and `vm_bo->list.gpuva`
  - the goal is to allow immediate `drm_gpuva_link` and `drm_gpuva_unlink`
    calls during vm map/unmap op which can be on the fence signaling path
    - `obj->resv` cannot be locked on the fence signaling path
    - because mem alloc is not allowed on the fence signaling path, a
      corollary is that mem alloc is not allowed while holding
      `obj->gpuva.lock`
  - see `drm_gpuvm_immediate_mode` and how `drm_gem_gpuva_assert_lock_held`
    changes its behavior
  - in older kernels, there was no `DRM_GPUVM_IMMEDIATE_MODE` nor
    `obj->gpuva.lock`
    - there was `drm_gem_gpuva_set_lock` which allowed the driver to specify
      an external lock that served the same purpose as `obj->gpuva.lock`
- locking during exec job or vm job submission
  - prealloc unique per-vm vm bo
    - `drm_gpuvm_bo_obtain_locked` creates vm bo and updates `obj->gpuva.list`
      - it expects `obj->resv` to be locked
    - alternative, if `DRM_GPUVM_IMMEDIATE_MODE`,
      - `drm_gpuvm_bo_create` creates a new vm bo
      - `drm_gpuvm_bo_obtain_prealloc` ensures a unique vm bo
        - it locks `obj->gpuva.lock`
    - `drm_gpuvm_bo_extobj_add` adds the vm bo to `vm->extobj.list`
      - it locks `vm->extobj.lock` by default
      - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
        locked instead
  - lock gem objs
    - `drm_gpuvm_prepare_vm` locks `vm->r_obj->resv`
    - `drm_gpuvm_prepare_objects` locks extobjs
      - it locks `vm->extobj.lock` by default
      - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
        locked instead
    - `drm_gpuvm_validate` validates evicted objs
      - it locks `vm->evict.lock` by default
      - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
        locked instead
      - `ops->vm_bo_validate` calls `drm_gpuvm_bo_evict(vm_bo, false)`
        - it locks `vm->evict.lock` by default
        - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
          locked instead
  - after the job is armed and there is an out-fence,
    - `drm_gpuvm_resv_add_fence` adds the out-fence to each `obj->resv`
      - it expects all gem jobs to be locked
  - unlock gem objs
    - `drm_exec_fini` unlocks all
- locking during vm job execution
  - driver holds an external lock for the entire job execution
  - step map
    - `drm_gpuva_map` adds vma to vm and updates `vm->rb.{tree,list}`
      - it expects an external lock
    - `drm_gpuva_link` adds vma to vm bo and updates `vm_bo->list.gpuva`
      - it expects `obj->resv`, or `obj->gpuva.lock` if
        `DRM_GPUVM_IMMEDIATE_MODE`, to be locked
  - step unmap
    - `drm_gpuva_unmap` removes vma from vm and updates `vm->rb.{tree,list}`
      - it expects an external lock
    - `drm_gpuva_unlink` removes vma from vm bo and updates `vm_bo->list.gpuva`
      - it expects `obj->resv`, or `obj->gpuva.lock` if
        `DRM_GPUVM_IMMEDIATE_MODE`, to be locked
    - `drm_gpuva_unlink_defer` removes vma from vm bo and updates `vm_bo->list.gpuva`
      - it requires `DRM_GPUVM_IMMEDIATE_MODE` abd locks `obj->gpuva.lock`
- locking during vm job cleanup
  - `drm_gpuvm_bo_put` destroys the bo if refcount reaches 0
    - it updates `vm->extobj.list` if extobj
      - it locks `vm->extobj.lock` by default
      - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
        locked instead
    - it updates `vm->evict.list` if evicted
      - it locks `vm->evict.lock` by default
      - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
        locked instead
    - it updates `obj->gpuva.list`
      - it expects `obj->resv`, or `obj->gpuva.lock` if
        `DRM_GPUVM_IMMEDIATE_MODE`, to be locked
  - `drm_gpuvm_bo_put_deferred` defer-destroys the bo if refcount reachs 0
    - it requires `DRM_GPUVM_IMMEDIATE_MODE` and locks `obj->gpuva.lock`
      - it updates `vm->extobj.list` only when possible
      - it updates `vm->evict.list` only when possible
      - it updates `obj->gpuva.list`
    - it updates `vm->bo_defer`
      - it is a lockless list
  - `drm_gpuvm_bo_deferred_cleanup` destroys defer-destroyed vm bos
    - if `DRM_GPUVM_RESV_PROTECTED`, it locks `vm->r_obj->resv`
      - it updates `vm->extobj.list` now
      - it updates `vm->evict.list` now
- shrinker reclaim
  - `drm_gem_lru_scan` locks `obj->resv`
  - if `DRM_GPUVM_IMMEDIATE_MODE`, the shrink callback locks `obj->gpuva.lock`
  - `drm_gem_for_each_gpuvm_bo` loops through `obj->gpuva.list`
    - it locks each `vm->r_obj->resv`
      - note how we lock `obj->resv, obj->gpuva.lock, vm->r_obj->resv` in order
        - in job submission, we lock `vm->r_obj->resv, obj->resv` in order
        - the order is inversed and can deadlock if not careful
      - `drm_gpuvm_bo_evict(vm_bo, true)` marks evicted and udpates `vm->evict.list`
        - it locks `vm->evict.lock` by default
        - if `DRM_GPUVM_RESV_PROTECTED`, it expects `vm->r_obj->resv` to be
          locked instead
      - `drm_gpuvm_bo_for_each_va` loops through `vm_bo->list.gpuva`
