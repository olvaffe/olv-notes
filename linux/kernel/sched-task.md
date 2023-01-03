Task
====

## Boot

- per-cpu `current_task` is statically initialized to statically initialized
  `init_task` defined in `init/init_task.c`
  - `comm` is `INIT_TASK_COMM` (swapper)
  - `active_mm` is `init_mm`
  - `thread_pid` is `init_struct_pid` w/ pid 0
- when `start_kernel` reaches `rest_init`, it spawns two threads
  - `kernel_init` first, which has pid 1
    - it will `execve("/sbin/init")` later
  - `kthreadd` second, which has pid 2
    - kthreadd will respond to future `kthread_create` requests
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
  - `smp_init`
    - `idle_threads_init` forks an idle thread (`swapper/n`) from `init_task`
      for each CPU.  PIDs are all 0.
    - `cpuhp_threads_init` registers `cpuhp_threads`
    - `bringup_nonboot_cpus` brings secondary CPUs from `CPUHP_OFFLINE` to
      `CPUHP_ONLINE`
      - `cpuhp_up_callbacks` invokes callbacks to bring the CPU up step by
      	step according to `cpuhp_hp_states`
      - `bringup_cpu` enables the CPU core.  On x86, this calls
      	`native_cpu_up`.  The enabled CPU starts from `start_secondary`, with
	task set to the idle task.  After initialization, it calls
	`cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)`
      - the last step calls `sched_cpu_activate` to tell the scheduler the CPU
      	is online
- the idle task on each CPU enters `cpu_startup_entry` and never exits
  - i.e., when it is scheduled on the CPU, the CPU goes idle

## `sys_mmap`

- see `mm`
