DRM XE
======

## Initialization

- `xe_irq_install` installs irq
  - `guc2host_irq_handler`
  - `xe_irq_msix_default_hwe_handler`
- `xe_gt_init` inits a GT
  - `xe_execlist_init` is nop if uc is enabled
  - `xe_hw_engines_init` inits engines
  - `xe_uc_init_post_hwconfig`
    - `xe_guc_init_post_hwconfig` calls `xe_guc_submit_init`
      - `gt->exec_queue_ops = &guc_exec_queue_ops;`

## IOCTLs

- `xe_exec_queue_create_ioctl` creates a queue
  - `xe_hw_engine_lookup` looks up the engine
  - `xe_exec_queue_create` creates the queue
  - `xe_lrc_create` creates a logical ring context
    - `lrc->bo` holds the ring buffer, sw states (e.g., seqno), etc.
  - `q->ops->init` is usually `guc_exec_queue_init`
    - `xe_sched_init` inits a scheduler with `drm_sched_ops`
      - the wq is `gt->ordered_wq`, an ordered wq with regular priority
    - `xe_sched_entity_init` inits a sched entity

## Jub Submission

- `xe_exec_ioctl`
  - `xe_sched_job_create` allocs a job
  - `xe_sched_job_arm` arms a job
    - `job->lrc_seqno`  is the lrc seqno
    - `job->fence` is a `dma_fence_chain` whose fences are initialized from
      `xe_lrc_init_seqno_fence`
  - `xe_sched_job_push` pushes the job
- `guc_exec_queue_run_job` is called when the job is ready to run
  - `q->ring_ops->emit_job` is `emit_job_gen12_render_compute`
    - `emit_bb_start` emits `MI_BATCH_BUFFER_START`
    - `emit_pipe_imm_ggtt` emits a pipelined write of the lrc seqno
    - `emit_user_interrupt` emits `MI_USER_INTERRUPT`
  - `job->fence` is returned as the hw fence
- on irq,
  - `xe_irq_msix_default_hwe_handler` calls `xe_hw_engine_handle_irq` with
    `GT_RENDER_USER_INTERRUPT`
  - `xe_hw_fence_irq_run` queues `hw_fence_irq_run_cb`
  - `hw_fence_irq_run_cb` signals the fence
