Linux workqueue
===============

## Overview

- there are system-wide worker pools
  - the number of workers in each pool increases and decreases automatically
    depending on the load
- `alloc_workqueue` allocates a workqueue
  - works queued via the workqueue will be dispatched to system-wide worker
    pools
  - workqueue attributes determine which worker pools to use
  - "pwq" is the glue that dispatches works to a worker pool
- `queue_work` queues a work to a queue

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
