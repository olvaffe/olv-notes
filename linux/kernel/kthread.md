Linux kthread
=============

## Boot

- `kernel_init`
  - after `workqueue_init`, workqueue queues start running which spawn a
    kthread each
    - `rcu_gp`, `rcu_par_gp`, etc.
  - `init_mm_internals` starts `mm_percpu_wq`
  - after `do_pre_smp_initcalls`, `early_initcall`s are executed.
    - `rcu_spawn_core_kthreads` registers `rcu_cpu_thread_spec`
    - `spawn_ksoftirqd` registers `softirq_threads`
    - `cpu_stop_init` registers `cpu_stop_threads`
    - `idle_inject_init` registers `idle_inject_threads`
    - `rcu_spawn_gp_kthread` starts `rcu_gp_kthread` (comm is `rcu_preempt`)

## Worker

- `kthread_create_worker` or `kthread_run_worker` creates a kthread worker
  - `kthread_worker_fn` is the thread entrypoint
    - it pops and execs `worker->work_list` in order
- `kthread_queue_work` queues a work
  - it adds the work to `worker->work_list` and wakes up the worker
