Kernel smp
==========

## Boot

- `smp_init` is called by the boot cpu
  - `idle_threads_init` forks an idle thread (`swapper/n`) from `init_task`
    for each CPU.  PIDs are all 0.
  - `cpuhp_threads_init` registers `cpuhp_threads`
  - `bringup_nonboot_cpus` brings secondary CPUs from `CPUHP_OFFLINE` to
    `CPUHP_ONLINE`
    - `cpuhp_up_callbacks` invokes callbacks to bring the CPU up step by
      step according to `cpuhp_hp_states`
    - `bringup_cpu` enables the CPU core with the help of arch-specific
      `__cpu_up`
      - On x86, this calls `native_cpu_up`.  The enabled CPU starts from
        `start_secondary`, with task set to the idle task.  After
        initialization, it calls `cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)`
    - the last step calls `sched_cpu_activate` to tell the scheduler the CPU
      is online
- the idle tasks on each CPU enter `cpu_startup_entry` and never exit
  - i.e., when it is scheduled on the CPU, the CPU goes idle
