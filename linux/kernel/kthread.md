Linux kthread
=============

## What are these kthreads?

- pid 0 enters `rest_init`
  - `user_mode_thread` forks off pid 1, starting in `kernel_init`
  - `kernel_thread` forks off pid 2, starting in `kthreadd`
- pid 1 enters `kernel_init`
  - `workqueue_init`
    - `kthread_run_worker` forks off `pool_workqueue_release`, starting in
      `kthread_worker_fn`
    - `init_rescuer` forks off `kworker/R-<name>` for each `WQ_MEM_RECLAIM` wq
  - `init_mm_internals`
    - `alloc_workqueue` forks off `kworker/R-mm_percpu_wq`
  - `do_pre_smp_initcalls` calls `early_initcall`s
    - `rcu_spawn_core_kthreads` registers `rcu_cpu_thread_spec`
    - `spawn_ksoftirqd` registers `softirq_threads`
    - `cpu_stop_init` registers `cpu_stop_threads`
    - `idle_inject_init` registers `idle_inject_threads`
    - `rcu_spawn_gp_kthread` starts `rcu_gp_kthread` (comm is `rcu_preempt`)
  - `driver_init`
    - `devtmpfs_init` creates `kdevtmpfs`
  - `do_initcalls` creates many more

## Worker

- `kthread_create_worker` or `kthread_run_worker` creates a kthread worker
  - `kthread_worker_fn` is the thread entrypoint
    - it pops and execs `worker->work_list` in order
- `kthread_queue_work` queues a work
  - it adds the work to `worker->work_list` and wakes up the worker
