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
  the highest priority and has all dependencies resolved

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
      - currently, the number of entities in runqueues
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
        spot but until dependencies are resolved
- `drm_sched_entity_push_job` submits the job to the entity

## Usage

- on job submission,
- job execution
  - `drm_sched_entity_pop_job` is called by the kthread to pop a job
    - if it returns a job, the job's dependencies have signaled
    - it also calls `drm_sched_backend_ops::prepare_job` to make sure there is
      no more dependency
  - `drm_sched_job_begin` adds the job to the scheduler and starts a timeout
    work
  - `drm_sched_backend_ops::run_job` queues the job to hw and returns a driver
    fence
  - `drm_sched_fence_scheduled` signals the `scheduled` fence
- on job completion,
  - driver irq signals the driver fence associated with the job
  - `drm_sched_job_done_cb` calls `drm_sched_job_done`
  - `drm_sched_fence_finished` signals the `finished` fence
- job cleanup
  - `drm_sched_get_cleanup_job` is also called from the scheduler to remove
    finished jobs from `drm_gpu_scheduler`
  - `drm_sched_backend_ops::free_job` frees the job by calling
    `drm_sched_job_cleanup`

## GPU Hang

- `drm_sched_run_job_work` calls `drm_sched_backend_ops::run_job` to queue the
  job to hw
  - before queuing, `drm_sched_start_timeout` starts `sched->work_tdr` with a
    timeout
- after job completion, `drm_sched_job_done` schedules
  `drm_sched_free_job_work`
  - `drm_sched_get_finished_job` cancels `sched->work_tdr`
- if the job does not complete in time, `drm_sched_job_timedout` is called
  - it calls `drm_sched_backend_ops::timedout_job` to trigger gpu recovery
- if the driver detects hangs before the timeout, it can call
  `drm_sched_fault` to schedule `drm_sched_job_timedout` immediately
