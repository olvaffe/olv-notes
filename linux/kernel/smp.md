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
    - on x86, this calls `cpuhp_kick_ap_alive` and `cpuhp_bringup_ap`
      - `native_kick_ap` calls `do_boot_cpu` to boot each secondary core
      - `secondary_startup_64` is the entrypoint
      - `idle` is from `idle_thread_get`, the idle thread
      - `initial_code` is set to `start_secondary`
      - `secondary_startup_64` calls `initial_code` at the end
      - `start_secondary` calls `cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)` at
        the end to enter the infinite idle loop
    - the last step calls `sched_cpu_activate` to tell the scheduler the CPU
      is online
- the idle tasks on each CPU enter `cpu_startup_entry` and never exit
  - i.e., when it is scheduled on the CPU, the CPU goes idle
