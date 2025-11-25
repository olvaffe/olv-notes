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
