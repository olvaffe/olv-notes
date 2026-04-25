# DRM exec

## Goal

- before job submission, drivers need to lock all bos to update their states
  atomically
- if there is a concurrent submit happening on another queue, and the
  concurrent submit needs to lock some common bos in a different order, the
  locking must be done carefully to avoid deadlock

## Usage

- `drm_exec_init` inits `drm_exec`
- `drm_exec_until_all_locked(&exec) { ... }` is the retry loop
  - it essentially expand to a jump label followed by
    `while (drm_exec_cleanup(&exec)) { ... }`
  - the loop body
    - calls `drm_exec_lock_obj` (and variants) to lock objects
    - optionally calls `drm_exec_retry_on_contention` to check for contention
      and restart the loop immediately
      - it works similar to `if (drm_exec_is_contended(&exec)) continue;`
  - `drm_exec_cleanup` returns false to exit the loop if all objects are locked
    - otherwise, it returns true after unlocking locked objects to retry the loop
- `drm_exec_fini` unlocks locked objects and cleans up `drm_exec`

## Internal

- `drm_exec_cleanup`
  - on first loop, it inits `exec->contended` and `exec->ticket`
  - on retries, it calls `drm_exec_unlock_all` to unlock locked objects
- `drm_exec_lock_obj` locks an object
  - `drm_exec_lock_contended` is an optimization to pre-lock `exec->contended`
    - if the loop contends with another loop, the first contended object is
      saved to `exec->contended`
    - upon retry, pre-locking `exec->contended` allows early fail
  - depending on `DRM_EXEC_INTERRUPTIBLE_WAIT`, `dma_resv_lock_interruptible`
    or `dma_resv_lock` locks the object
  - if `-EALREADY` and `DRM_EXEC_IGNORE_DUPLICATES`, it is considered locked
  - otherwise, the locking fails
    - if `-EDEADLK`, `exec->contended` is updated
- `drm_exec_prepare_obj` is `drm_exec_lock_obj` plus `dma_resv_reserve_fences`
