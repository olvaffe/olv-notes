DRM Scheduler
=============

## Overview

- there is a `drm_gpu_scheduler` per-hw-ring to schedule jobs to
- each `drm_gpu_scheduler` has 1 or more runqueues, `drm_sched_rq`, with
  different priorities
- userspace creates `drm_sched_entity`s with different priorities
- userspace submits `drm_sched_job` to `drm_sched_entity`
  - this dynamically moves the entity to the runqueue of the best
    `drm_gpu_scheduler`
   - e.g., when the hw has multiple compute rings, it picks the least busy one
- when `drm_gpu_scheduler` schedules a job to hw ring, it picks a job that has
  the highest priority and has all deps resolved

## Initialization

- `drm_sched_init` inits a `drm_gpu_scheduler` per hw ring
  - params
    - `ops` is `drm_sched_backend_ops`
    - `submit_wq` is the submit workqueue
      - it can be ordered (one work at a time) or not
    - `num_rqs` is the number of `drm_sched_rq`, from high to low priorities
    - `credit_limit` is a u32, which is typically ring size
    - `hang_limit` is deprecated (used by deprecated `drm_sched_resubmit_jobs`)
    - `timeout` is number of jiffies before a running job is timed out
    - `timeout_wq` is the timeout workqueue, typically `system_wq`
    - `score` is the busyness of the scheduler
      - currently, the number of entities in runqueues plus jobs in entities
        or running
  - `sched_rq` is an array of `drm_sched_rq`, from high to low priorities
  - `drm_sched_job_timedout`
  - `drm_sched_run_job_work`
  - `drm_sched_free_job_work`
- `drm_sched_entity_init` inits a `drm_sched_entity` per userspace queue
  - params
    - `priority` is the priority
    - `sched_list` and `num_sched_list` are an array of `drm_gpu_scheduler`
      - `drm_sched_job_arm` will call `drm_sched_entity_select_rq` to move the
        entity to the best scheduler dynamically, if there are more than one
        scheduler
    - `guilty` will be set to 1 when the entity has a hanging job
  - `fence_context` has 2 fence contexts, for `scheduled` and `finished`

## Job Submission

- `drm_sched_job_init` initializes a job
  - params
    - `entity` is the entity to submit to
    - `credits` is a u32, which is typically instr count
    - `owner` is saved in `drm_sched_fence`
  - `s_fence` is a `drm_sched_fence`
  - `dependencies` is a `xarray` of `dma_fence`
- `drm_sched_job_add_dependency` adds an explicit in-fence to
  `job->dependencies`
- `drm_sched_job_add_syncobj_dependency` calls `drm_sched_job_add_dependency`
  on the `dma_fence` fished out from an explicit explicit in-syncobj
- `drm_sched_job_add_resv_dependencies` calls `drm_sched_job_add_dependency`
  on `dma_fence`s in a `dma_resv`
- `drm_sched_job_add_implicit_dependencies` calls
  `drm_sched_job_add_resv_dependencies` on `obj->resv` to add implicit fences
- `drm_sched_job_arm` arms a job
  - `drm_sched_entity_select_rq` selects the best scheduler, if the entity has
    multiple schedulers
    - `drm_sched_pick_best` picks the scheduler with the lowest `score`
    - `drm_sched_rq_remove_entity` removes the entity from the old rq
    - `entity->rq` is set to the new rq
  - `drm_sched_fence_init` inits `job->s_fence`
    - `dma_fence_init` inits `scheduled` and `finished` fences
    - if the driver needs an out-fence for the submit, it should use
      `finished`
      - `run_job` will return the hw out-fence, but it is not called on the
        spot but until deps are resolved
- `drm_sched_entity_push_job` submits the job to the entity
  - the job is pushed to `entity->job_queue`
  - if the job queue is empty,
    - `drm_sched_rq_add_entity` adds the entity to `rq->entities`
    - `drm_sched_rq_update_fifo_locked` adds the entity to `rq->rb_tree_root`
    - `drm_sched_wakeup` queues `drm_sched_run_job_work`

## Job Execution

- `drm_sched_run_job_work` picks the next job and submits it to hw ring
- `drm_sched_select_entity` selects an entity that is potentially ready
  - it calls `drm_sched_rq_select_entity_fifo` on runqueues, from high to low
    priorities, and returns the first entity that is ready
    - it loops through entities on `rq->rb_tree_root`, which is sorted by
      `entity->oldest_job_waiting`
    - `drm_sched_entity_is_ready` returns true if the entity is potentially
      ready
    - `drm_sched_can_queue` checks if the scheduler has enough credits (ring
      space) for the job
- `drm_sched_entity_pop_job` pops the first job from the entity, if it is
  ready
  - `drm_sched_entity_queue_peek` peeks the first job
  - `drm_sched_job_dependency` returns the next dep in
    `job->dependencies`
    - if driver has deps not in `job->dependencies`, it can provide
      `sched->ops->prepare_job` to return those deps
  - `drm_sched_entity_add_dependency_cb` checks if a dep requires waiting
    - if the fence is from another job of the same entity, no wait needed
      because jobs of the same entity are executed in order
    - if the fence is from another job of the same scheduler, wait for
      `scheduled` to call `drm_sched_entity_clear_dep`
      - this clears `entity->dependency` and marks the entity potentially
        ready
    - otherwise, wait for the fence to call `drm_sched_entity_wakeup`
      - this addtionally calls `drm_sched_wakeup`
  - it early returns if the first job has any dep that requires waiting
  - `entity->last_scheduled` is set to `sched_job->s_fence->finished`
    - this will be used to check if the last scheduled job has finished
  - `drm_sched_rq_update_fifo_locked` ensures entities in the runqueue is
    sorted, to make `drm_sched_select_entity` work
- it adds `job->credits` to `sched->credit_count`
- `drm_sched_job_begin` adds the job to `sched->pending_list` and starts
  timeout timer
- `sched->ops->run_job` submits the job to hw ring and returns the hw fence
- `entity->entity_idle` means that a entity is not being used, mainly by
  `drm_sched_entity_pop_job`, and `drm_sched_entity_kill` can kill it
- `drm_sched_fence_scheduled`
  - `drm_sched_fence_set_parent` makes the hw fence the parent
  - it signals `scheduled`, now that the job has been submitted to hw ring
- it sets up `drm_sched_job_done_cb` to be called when the hw fence signals
- it queues `drm_sched_run_job_work` again

## Job Completion

- `drm_sched_job_done_cb` is called when the hw fence of the job signals
  - it subtracts `job->credits` from `sched->credit_count`
  - it decrements `sched->score`
  - `drm_sched_fence_finished` signals `finished`
  - it queues `drm_sched_free_job_work`
- `drm_sched_free_job_work`
  - `drm_sched_get_finished_job` pops the first job on `sched->pending_list`
    if it has signaled
    - it also cancels the current timeout timer, and starts timeout timer for
      the next job
  - `sched->ops->free_job` frees the job
  - `drm_sched_run_free_queue` queues `drm_sched_free_job_work` again if the
    next job has also signaled
  - `drm_sched_run_job_queue` queues `drm_sched_run_job_work`

## Job Timeout

- `drm_sched_run_job_work`
  - `drm_sched_job_begin` starts timeout timer
  - `sched->ops->run_job` submits the job to hw ring
- if the job completes before the timeout timer,
  - the hw fence calls `drm_sched_job_done_cb` to signal `finished` fence and
    queues `drm_sched_free_job_work`
  - `drm_sched_free_job_work` cancels the timeout timer
- if the job times out, `drm_sched_job_timedout` is called
  - it pops the first job from `sched->pending_list`
  - `sched->ops->timedout_job` cancels the job
- if the driver detects hangs before the timeout, it can call
  `drm_sched_fault` to schedule `drm_sched_job_timedout` immediately
- a typical `timedout_job` impl
  - `drm_sched_stop` stops the scheduler
    - `drm_sched_wqueue_stop` cancels works and sets `pause_submit`
    - it pushes the job back to `sched->pending_list`
    - it loops through `sched->pending_list` to either remove fence callbacks
      (if unsignaled) or to wait for fence callbacks (if signaled)
    - it cancels timeout timer
  - cancel the bad job from hw somehow
    - if supported, remove the bad job directly
    - otherwise, pause all good jobs, reset gpu, and resume all good jobs
    - note that all hw fences, including the hw fence for the bad job, must
      still signal in finite time
  - `drm_sched_start` starts the scheduler
    - it loops through `sched->pending_list` to set up `drm_sched_job_done_cb`
      again
    - `drm_sched_start_timeout_unlocked` starts timeout timer
    - `drm_sched_wqueue_start` resets `pause_submit` and queues works

## Fence Contract

- <https://docs.kernel.org/driver-api/dma-buf.html#dma-fence-cross-driver-contract>
  - a driver is allowed to call `dma_fence_wait`
    - while holding `dma_resv_lock`,
    - from memory shrinker, or
    - from `mmu_interval_notifier`
  - the same driver, or another driver, must avoid these on the code path
    leading to `dma_fence_signal` to avoid deadlock
    - `dma_resv_lock`
    - allocation with `GFP_KERNEL`, `GFP_NOFS`, or `GFP_NOIO`
      - effectively, only `GFP_ATOMIC` is allowed
- after `drm_sched_run_job_work`, the hw fence must signal in finite time
  - if the job completes, the hw raises an irq and the driver calls
    `dma_fence_signal` (directly, or indirectly via `dma_fence_is_signaled`)
    in its irq handler
    - the irq handler is the code path leading to `dma_fence_signal`
  - if the job times out and is canceled by `drm_sched_job_timedout`, the
    driver still needs to call `dma_fence_signal` one way or another
    - the `timedout_job` callback is on the path leading to `dma_fence_signal`
    - if `timedout_job` merely requests the hw to cancel, and a cancel irq
      handler signals the fence, the irq handler is also on the path leading
      to `dma_fence_signal`
