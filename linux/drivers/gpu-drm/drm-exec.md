DRM exec
========

## `drm_exec`

- before job submission, drivers need to lock all bos to update their states
  atomically
- if there is a concurrent submit happening on another queue, and the
  concurrent submit needs to lock some common bos in a different order, the
  locking must be done carefully to avoid deadlock
- `drm_exec_init` prepares `drm_exec`
- `drm_exec_until_all_locked(&exec){...;drm_exec_retry_on_contention(&exec)}`
  is the retry loop
  - `drm_exec_until_all_locked` essentially expands to a jump label followed
    by `while (drm_exec_cleanup(&exec))`
  - `drm_exec_retry_on_contention` expands to a jump to the label if
    contention happens
  - `drm_exec_cleanup` returns false only when all objects have been locked
    - otherwise, it unlocks all objects and returns true to restart
- inside the retry loop is usually another loop to lock objects one-by-one
  - `drm_exec_prepare_obj` calls `drm_exec_lock_obj` and
    `dma_resv_reserve_fences`
- `drm_exec_fini` unlocks all objs and cleans up `drm_exec`
