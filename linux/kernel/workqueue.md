Linux workqueue
===============

## Overview

- there are system-wide worker pools
  - each cpu gets 4 worker pools
    - `init_cpu_worker_pool` inits a worker pool
    - `kthread_set_per_cpu` confines the worker to `pool->cpu` (not -1)
  - there are also unbound worker pools
    - `get_unbound_pool` creates one on demand
  - the number of workers in each pool increases and decreases automatically
    depending on the load
    - `maybe_create_worker` and `idle_cull_fn`
- `alloc_workqueue` allocates a workqueue
  - works queued via the workqueue will be dispatched to system-wide worker
    pools
  - workqueue attributes determine which worker pools to use
  - `alloc_and_link_pwqs` creates pwqs
    - if `WQ_UNBOUND`, each per-cpu pwq points to a `get_unbound_pool` pool
    - if `WQ_PERCPU`, each per-cpu pwq points to the respective per-cpu worker pool
- `queue_work` queues a work to a queue
  - it looks up the per-cpu pwq, which determines the worker pool
  - the work is added to `pool->worklist`, or if too many active works,
    `pwq->inactive_works`
- the purpose of a worker pool is unlike a typical thread pool
  - a typical thread pool aims to saturate multiple cores
  - a worker pool aims to saturate a single core
  - when work1 and work2 are both queued to the same worker pool with worker1 and worker 2,
    - if work1 does not sleep, worker1 typically executes both work1 and work2 in order
    - if work1 sleeps, worker1 executes work1 until it sleeps; at that point,
      worker2 wakes up to execute work2
  - `WQ_UNBOUND` is special
    - it uses pools from `get_unbound_pool` which have `POOL_DISASSOCIATED`
    - the pool workers have `WORKER_UNBOUND`, which is considered `WORKER_NOT_RUNNING`
      - see the comment of `need_more_worker`
    - when N works are queued in a row, it can spawn up to N-1 workers

## Flags

- `max_active` defaults to `WQ_DFL_ACTIVE` (1024)
  - `wq_adjust_max_active`
    - if `WQ_UNBOUND`, the limit is per-node
    - if `WQ_PERCPU`, the limit is per-cpu
  - `__queue_work` calls `pwq_tryinc_nr_active` to determine whether a work
    goes to `pool->worklist` or `pwq->inactive_works`
    - if `WQ_UNBOUND`, `tryinc_node_nr_active` increments the per-node `nna->nr`
    - if `WQ_PERCPU`, it increments the per-cpu `pwq->nr_active`
- flags
  - `WQ_BH`
  - `WQ_UNBOUND` will become the default
  - `WQ_FREEZABLE`
    - `freeze_workqueues_begin` calls `wq_adjust_max_active` to set `max_active`
      to 0
  - `WQ_MEM_RECLAIM` allows the wq to be used during mem reclaim
    - `init_rescuer` spawns a rescuer kthread for the wq
    - when `maybe_create_worker` creates a worker, but can't due to oom,
      mayday timer moves the pwq to `wq->maydays` and execute the works on the
      rescuer kthread
  - `WQ_HIGHPRI` uses worker pools who workers are `HIGHPRI_NICE_LEVEL`
    - `alloc_and_link_pwqs` picks highpri pools
  - `WQ_CPU_INTENSIVE`
    - `need_more_worker` returns true if there are works on `pool->worklist`
      and `pool->nr_running` is 0 (no running worker)
    - when a worker executes a cpu intensive work, it is excluded from `pool->nr_running`
      - see `worker_set_flags` and `WORKER_NOT_RUNNING`
  - `WQ_SYSFS` exposes `/sys/devices/virtual/workqueue/<wq>` for config
  - `WQ_POWER_EFFICIENT` is nop by default
    - admin can configure it to force `WQ_UNBOUND`
  - `WQ_PERCPU` is the opposite of `WQ_UNBOUND`
  - `__WQ_ORDERED` is used with `WQ_UNBOUND` to share `ctx->dfl_pwq`
- `WQ_UNBOUND`, `WQ_PERCPU`, and `__WQ_ORDERED`
  - `alloc_and_link_pwqs`
    - if `WQ_PERCPU`, per-cpu pwqs point to per-cpu worker pools
    - if `WQ_UNBOUND`, per-cpu pwqs point to the same (unless numa) unbound worker pool
      - if `__WQ_ORDERED`, per-cpu pwqs actually are the same `ctx->dfl_pwq`
        pointing to an unbound worker pool
  - when `queue_work` is called from cpu N,
    - if `WQ_PERCPU`, per-cpu pwq N dispatches the work to per-cpu pool N
    - if `WQ_UNBOUND`, per-cpu pwq N dispatches the work to an unbound worker pool
      - if `__WQ_ORDERED`, `ctx->dfl_pwq` dispatches the work to an unbound worker pool
  - `alloc_ordered_workqueue` is `WQ_UNBOUND | __WQ_ORDERED` and `max_active == 1`
    - all works are queued to the same pwq
    - if a work is running, the remaining works are parked on `pwq->inactive_works`

## Initialization

- `workqueue_init_early` is called early from `start_kernel`
  - it inits various global variables
  - it inits `wq_pod_types[WQ_AFFN_SYSTEM]`
    - `nr_pods` is 1
    - `pod_cpus` is `cpu_possible_mask` (all cpus)
    - `pod_node` is `NUMA_NO_NODE`
    - `cpu_pod` is 0
  - it inits `bh_worker_pools` and `cpu_worker_pools`
    - each cpu has 4 `worker_pool`s, `(cpu, bh) x (normal, highprio)`
  - it allocs a bunch of global workqueues
    - `system_wq`
    - `system_highpri_wq` with `WQ_HIGHPRI`
    - `system_long_wq`
    - `system_unbound_wq` with `WQ_UNBOUND` and `WQ_MAX_ACTIVE`
    - `system_freezable_wq` with `WQ_FREEZABLE`
    - `system_power_efficient_wq` with `WQ_POWER_EFFICIENT`
    - `system_freezable_power_efficient_wq` with `WQ_FREEZABLE | WQ_POWER_EFFICIENT`
    - `system_bh_wq` with `WQ_BH`
    - `system_bh_highpri_wq` with `WQ_BH | WQ_HIGHPRI`
  - after this returns,
    - `alloc_workqueue` can allocate workqueues
    - `queue_work` can queue works
    - but there is no worker to execute works
- `workqueue_init` is called from `kernel_init_freeable` before `smp_init`
  - `create_worker` create a worker for each of the `worker_pool`s
    - there are
      - `bh_worker_pools`
      - `cpu_worker_pools`
      - `unbound_pool_hash`
    - `alloc_worker` allocs a `worker`
    - if the pool is not `POOL_BH`, `kthread_create_on_node` creates a kthread
      executing `worker_thread`
      - otherwise, it is executed in softirq context
    - `worker_attach_to_pool` attaches the worker to the pool
    - `worker_enter_idle` marks the worker idle
  - `wq_online` is set to true
- `workqueue_init_topology` is called from `kernel_init_freeable` after
  `smp_init`
  - other `wq_pod_types` are initialized
  - `unbound_wq_update_pwq` allocs `pool_workqueue` for `WQ_UNBOUND` queues

## Worker Pools

- workers run `worker_thread`
- `manage_workers` may spawn more workers if necessary
- worker takes the first work from `pool->worklist`
- `assign_work` moves the work to `worker->scheduled`
- `process_scheduled_works` loops through `worker->scheduled` and calls
  `process_one_work`
  - `set_work_pool_and_clear_pending` clears `WORK_STRUCT_PENDING_BIT`
    - note that the work can be queued again after this point, even before it
      starts running
  - `worker->current_func(work)` calls the work function

## Work

- a `struct work_struct` is very compact
  - `atomic_long_t data` has various meanings
  - `struct list_head entry` is for `insert_work` to add the work to
    `pool->worklist`, `pwq->inactive_works`, etc.
  - `work_func_t func` is the callback
- `atomic_long_t data`
  - bit layout
    - bit 0..3
      - `WORK_STRUCT_PENDING_BIT`
      - `WORK_STRUCT_INACTIVE_BIT`
      - `WORK_STRUCT_PWQ_BIT`
      - `WORK_STRUCT_LINKED_BIT`
    - if `WORK_STRUCT_PWQ_BIT`,
      - bit 4..7: `WORK_STRUCT_COLOR_BITS`
      - bit 8..: pwq pointer (aligned to 256 bytes)
    - if no `WORK_STRUCT_PWQ_BIT`,
      - bit 4: `WORK_OFFQ_BH_BIT`
      - bit 5..20: `WORK_OFFQ_DISABLE_BITS`
      - bit 21..: `WORK_OFFQ_POOL_BITS` (pool id)
  - `INIT_WORK` inits data to `WORK_DATA_INIT`
    - pool id is `WORK_OFFQ_POOL_NONE`
    - the rest is zero
  - when `queue_work` queues a work,
    - `test_and_set_bit` sets data to
      - `WORK_STRUCT_PENDING`
    - when `insert_work` adds the work to a list, `set_work_pwq` sets data to
      - `WORK_STRUCT_PENDING`
      - `WORK_STRUCT_PWQ`
      - color bits
      - pwq pointer
  - when `process_one_work` processes the work,
    `set_work_pool_and_clear_pending` sets data to
    - pool id
  - when `work_grab_pending` steals a work from pwq for modifications,
    `set_work_pool_and_keep_pending` sets data to
    - `WORK_STRUCT_PENDING`
    - pool id

## API

- `alloc_workqueue` allocs a `workqueue_struct`
  - `flags`
    - `WQ_BH`
    - `WQ_UNBOUND`
    - `WQ_FREEZABLE`
    - `WQ_MEM_RECLAIM`
    - `WQ_HIGHPRI`
    - `WQ_CPU_INTENSIVE`
  - `max_active` is the max active work items per-cpu
- `INIT_WORK` inits a `work_struct`
- `queue_work` queues a work to a queue
  - if `WORK_STRUCT_PENDING_BIT`, returns false
  - `insert_work` adds the work to `pool->worklist` or `pwq->inactive_works`
  - `kick_pool` wakes up an idle worker if needed
- a work item is executed by at most one worker system-wide at any given time,
  if
  - the work function is not changed,
  - the item is not queued to another queue, and
  - the item is not re-initialized

## `delayed_work`

- a `delayed_work` is a work with a timer
- `INIT_DELAYED_WORK` inits a delayed work with the specified fn
  - `INIT_WORK` inits `work` with the specified fn
  - `__timer_init` inits `timer` with `delayed_work_timer_fn`
- to queue a delayed work,
  - `queue_delayed_work`
    - `WORK_STRUCT_PENDING_BIT` means the work is pending execution
      - that is, it is queued and is not disabled
    - if the work is already pending,
      - if the work is enabled, nop
      - if the work is disabled, `clear_pending_if_disabled` clears the bit
    - if the work is not pending,
      - the bit is set
      - `__queue_delayed_work` queues the delayed work
        - if there is delay, `add_timer_global` starts the timer
        - if there is no delay, `__queue_work` queues the work
  - `schedule_delayed_work` is `queue_delayed_work` on `system_percpu_wq`
  - `mod_delayed_work`
    - `work_grab_pending` steals the work for modification
      - if there is a timer, `timer_delete` stops the timer
    - if not disabled, `__queue_delayed_work` queues the delayed work
- to flush (wait for completion if queued) a delayed work,
  - `flush_delayed_work`
    - `timer_delete_sync` stops the timer
      - it returns true only if the timer was started
      - `_sync` means, if the timer was fired, block until the handler returns
    - if the timer was started, `__queue_work` queues the work now
    - `flush_work` waits for work completion
- to cancel a delayed work,
  - `cancel_delayed_work`
    - `work_grab_pending` steals the work for modification
      - if there is a timer, `timer_delete` stops the timer
    - no queuing the work back
  - `cancel_delayed_work_sync`
    - it cancels and flushes the work
- to enable/disable a delayed work,
  - `disable_delayed_work` is `cancel_delayed_work` plus incrementing disable
    count
  - `disable_delayed_work_sync` is `cancel_delayed_work_sync` plus
    incrementing disable count
  - `enable_delayed_work` decrements disable count (only if the work is
    already disabled)
