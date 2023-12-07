DRM Scheduler
=============

## Usage

- data structures
  - there should be a `struct drm_gpu_scheduler` for each ring
  - there should be a `struct drm_sched_entity` for each priority for each
    opened file
- during device initialization, `drm_sched_init` should be called on each
  `struct drm_gpu_scheduler` of each ring
  - this spawns a kthread
- when a VkDevice is initialized and a VkQueue is created,
  `drm_sched_entity_init` should be called for each VkQueue
- on job submission,
  - `drm_sched_job_init` initializes a job
  - `drm_sched_job_add_dependency` adds explicit in-fences to the job
  - `drm_sched_job_add_implicit_dependencies` adds implicit fences to the job
    - this extracts the implicit fences from the bo
  - `drm_sched_job_arm` arms a job for execution
    - this initializes `scheduled` and `finished` fences; they will complete
      in finite time
  - `drm_sched_entity_push_job` submits the job to the entity
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
