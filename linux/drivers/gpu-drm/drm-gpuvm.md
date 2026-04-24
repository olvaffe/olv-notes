DRM GPUVM
=========

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

## How a Driver Uses GPUVM

- assume an op that
  - waits N syncobjs
  - operates on gem objs
  - signals M syncobjs
- create a job for the op
  - `drm_sched_job_init` inits the job
  - `drm_sched_job_add_syncobj_dependency` adds each of the N syncobj to wait
    as deps
  - collect M syncobjs to signal
    - `drm_syncobj_find` looks up the syncobj
    - if timeline, `dma_fence_chain_alloc` allocs a fence chain
- lock gem objs
  - `drm_exec_init` with `DRM_EXEC_IGNORE_DUPLICATES`
  - `drm_exec_until_all_locked` is the retry loop
    - `drm_gpuvm_prepare_vm` locks `vm->r_obj->resv`
    - `drm_gpuvm_prepare_objects` locks extobjs
    - `drm_exec_retry_on_contention` retries on contention
  - no pinning needed with gpuvm
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
  - `resv` can point to local `_resv` or `vm->r_obj->resv`
    - if an obj is exclusive to a vm, set `obj->resv` to `vm->r_obj->resv`
  - `gpuva.list` is the list of per-vm `drm_gpuvm_bo`
    - it ensure each obj has a unique vm bo in each vm
    - it is protected by `resv` by default
  - `gpuva.lock` is unsed by default
- `drm_gpuvm`
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
  - `list.gpuva` is the list of `drm_gpuva`
    - it is protected by `obj->resv` by default
  - `entry.gem` is for `obj->gpuva.list`
  - `entry.extobj` is for `vm->extobj.list`
  - `entry.evict` is for `vm->evict.list`
  - `entry.bo_defer` is for `vm->bo_defer`
- `drm_gpuva`
  - `rb.node` is for `vm->rb.tree`
  - `rb.entry` is for `vm->rb.list`
  - `gem.entry` is for `vm_bo->list.gpuva`
- if `DRM_GPUVM_RESV_PROTECTED`, `vm->extobj.lock` and `vm->evict.lock` become
  unused
  - driver must lock `vm->r_obj->resv` externally to protect `vm->extobj.list`
    and `vm->evict.list`
  - see `drm_gpuvm_resv_protected` and `drm_gpuvm_resv_assert_held`
- if `DRM_GPUVM_IMMEDIATE_MODE`, `obj->gpuva.lock` replaces `obj->resv` to
  protect `obj->gpuva.list`, `vm_bo->list.gpuva`, and `vm->bo_defer`
  - see `drm_gpuvm_immediate_mode` and how `drm_gem_gpuva_assert_lock_held`
    changes its behavior
