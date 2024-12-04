Mesa PanVK Job Manager
======================

## Queue Submit

- there can be multiple cmdbufs
  - `cmdbuf->batches` are the batches in a cmdbuf
  - `batch->jobs` are the jobs in a batch
    - it looks like each job is a cpu pointer to a hw job descriptor
    - each hw job descriptor starts with a `MALI_JOB_HEADER` which can chain
      to the next hw job descriptor
- `panvk_queue_submit`
  - it collects all syncobjs to wait
  - for each batch of each cmdbuf
    - it collects all bos referenced
    - it collects all events to wait
    - `panvk_queue_submit_batch` submits a batch and signals `queue->sync`
    - it transfers `queue->sync` to all events to signal
  - it transfers `queue->sync` to all syncobjs to signal
- `struct drm_panfrost_submit` describes a batch submit ioctl
  - `jc` is the gpu va of the first job descriptor
  - `in_syncs` are syncobjs to wait
  - `out_sync` is the syncobj to signal
  - `bo_handles` are bos to keep resident
  - `requirements` are flags
    - `PANFROST_JD_REQ_FS` selects slot 0 instead of slot 1
    - `PANFROST_JD_REQ_CYCLE_COUNT` enables cycle counter
      - needed when the batch writes timestamps
- kmd `panfrost_job_run` executes a job (a job chain actually, aka a batch)
  - `panfrost_job_get_slot` picks slot 0 for frag and slot 1 for the rest
    - slots are like hw queues and there are a total of 3 slots
    - each slot has 2 subslots, meaning only two batches can be queued
  - the va of the first job descriptor is written to `JS_HEAD_NEXT_LO` /
    `JS_HEAD_NEXT_HI`
  - the config is written to `JS_CONFIG_NEXT`
    - `panfrost_mmu_as_get` selects the address space
    - `JS_CONFIG_START_FLUSH_CLEAN_INVALIDATE` flushes/invalidates caches
      before exec
    - `JS_CONFIG_END_FLUSH_CLEAN_INVALIDATE` flushes/invalidates caches after
      exec
    - `JS_CONFIG_THREAD_PRI` selects the priority
  - `JS_COMMAND_START` is written to `JS_COMMAND_NEXT`
- kmd `panfrost_job_handle_irq` handles job completion (job chain completion)
  - `panfrost_job_handle_done` marks `job->done_fence` signaled

## Command Buffer Batches and Jobs

- batches
  - `panvk_per_arch(cmd_open_batch)` allocs `cmdbuf->cur_batch`
  - `panvk_per_arch(cmd_close_batch)` moves the current batch to
    `cmdbuf->batches` and resets `cmdbuf->cur_batch`
    - when ending a render pass, it adds frag jobs automatically
- jobs
  - `pan_jc_add_job` adds a job descriptor to the job chain
  - `MALI_JOB_TYPE_NULL` is a nop job, useful for empty queue submit
  - `MALI_JOB_TYPE_WRITE_VALUE` writes a value to an addr
    - the value can be cycle count, timestamp, zero, or any 8/16/32/64-bit val
  - `MALI_JOB_TYPE_CACHE_FLUSH` is unused
  - `MALI_JOB_TYPE_COMPUTE` is a compute job
  - `MALI_JOB_TYPE_VERTEX` is a legacy vertex-only job
  - `MALI_JOB_TYPE_TILER` is a legacy tiler-only job
  - `MALI_JOB_TYPE_FRAGMENT` is a frag job
  - `MALI_JOB_TYPE_INDEXED_VERTEX` is an idvs job for v6 and v7
  - `MALI_JOB_TYPE_MALLOC_VERTEX` is an idvs job for v9
- render pass
  - `panvk_per_arch(CmdBeginRendering)` opens a new batch
  - each `panvk_cmd_draw` adds a job to `vtc_jc`
    - if there are too many draws exceeding the max jobs, it closes the
      current batch and opens a new one
  - `panvk_per_arch(CmdEndRendering)` closes the current batch
    - it also adds a job to `frag_jc` for each layer before closing
- compute dispatch
  - `panvk_per_arch(CmdDispatchBase)` opens a new batch, adds a job to
    `vtc_jc`, and closes the batch
- barrier
  - `panvk_per_arch(CmdPipelineBarrier2)` closes the current batch and opens a
    new one
- event
  - event ops are handled similar to sempahores during queue submit
    - as a result, they must be at the beginning or at the end of a batch
  - `panvk_per_arch(CmdSetEvent2)` and `panvk_per_arch(CmdResetEvent2)` adds
    event ops to `cmdbuf->cur_batch->event_ops`, and closes the batch to make
    sure they are executed at the end of the batch on queue submit
  - `panvk_per_arch(CmdWaitEvents2)` opens a new batch and adds event ops to
    `cmdbuf->cur_batch->event_ops`, to make sure they are executed at the
    beginning of the batch on queue submit
