Kernel CPU
==========

## Boot CPU

- the boot cpu is always online, but we need to update sw state to match the
  hw state
- PID 0: `start_kernel`
  - `boot_cpu_init`
    - `set_cpu_online` sets `__cpu_online_mask`
    - `set_cpu_active` sets `__cpu_active_mask`
    - `set_cpu_present` sets `__cpu_present_mask`
    - `set_cpu_possible` sets `__cpu_possible_mask`
  - `setup_per_cpu_areas` inits per-cpu areas
    - on x86, per-cpu variables are accessed by `%gs:var`
      - `MSR_GS_BASE` is initialized to 0
      - all per-cpu accesses access the same instance in `.data..percpu` section
      - this allocs per-cpu areas
      - `switch_gdt_and_percpu_base` updates `MSR_GS_BASE` to offsets between
        the per-cpu areas and `.data..percpu` section
        - this is done for the boot cpu only
        - `secondary_startup_64` inits `MSR_GS_BASE` directly for secondary cpus
      - future per-cpu accesses access the per-cpu areas
  - `smp_prepare_boot_cpu` is arch-specific
  - `boot_cpu_hotplug_init`
    - it sets `cpus_booted_once_mask`
    - it sets `cpuhp_state.ap_sync_state` to `SYNC_STATE_ONLINE`
    - it sets `cpuhp_state.state` to `CPUHP_ONLINE`
    - it sets `cpuhp_state.target` to `CPUHP_ONLINE`
  - `arch_cpu_finalize_init` is arch-specific
  - `cpu_startup_entry` enters the idle loop
- PID 1: `kernel_init`
  - `smp_prepare_cpus` is arch-specific
  - `smp_init`
    - `cpuhp_threads_init`
      - `cpuhp_init_state` inits per-cpu `cpuhp_state`
      - `smpboot_register_percpu_thread` creates a `cpuhp/%u` thread for each
        cpu
    - `bringup_nonboot_cpus` brings up secondary cpus

## Secondary CPUs

- secondary cpus need to be brought online
- `smp_init` calls `bringup_nonboot_cpus`
  - with `CONFIG_HOTPLUG_PARALLEL`,
    - `cpuhp_bringup_mask(CPUHP_BP_KICK_AP)` calls `cpu_up(CPUHP_BP_KICK_AP)`
      for each secondary cpu
      - this boots all secondary cpus but leaves them spinning inside
        `cpuhp_ap_sync_alive`
    - `cpuhp_bringup_mask(CPUHP_ONLINE)` calls `cpu_up(CPUHP_ONLINE)` for each
      secondary cpu
      - this allows secondary cpus to go on to complete the boot

## CPU Hotplug

- `enum cpuhp_state`
  - secondary cpu is offline from `CPUHP_OFFLINE` to `CPUHP_BRINGUP_CPU`
    - boot cpu invokes callbacks for these states
  - secondary cpu is booting until `CPUHP_AP_ONLINE`
    - secondary cpu invokes callbacks for these states in idle thread context
  - secondary cpu is starting until `CPUHP_ONLINE`
    - secondary cpu invokes callbacks for these states in cpuhp thread context
  - note that the boot cpu itself does not go through the states
    - it skips to `CPUHP_ONLINE`
    - if a subsystem has a per-cpu callback, it must be called for the boot
      cpu manually
- `enum cpuhp_sync_state` is used to sync between boot cpu and secondary cpu
  as the state of the secondary cpu changes
- `cpu_up` is called by `bringup_nonboot_cpus` (or on userspace cpu up)
  - `cpuhp_set_state` sets `st->target` and `st->bringup`
  - `cpuhp_up_callbacks` brings a cpu up to `CPUHP_BRINGUP_CPU` first
    - `CPUHP_CREATE_THREADS` calls `smpboot_create_threads`
      - it creates but parks per-cpu threads registered by
        `smpboot_register_percpu_thread`, such as `cpuhp/%u`, `ksoftirqd/%u`,
        `migration/%u`, etc.
    - `CPUHP_RANDOM_PREPARE` calls `random_prepare_cpu`
    - `CPUHP_WORKQUEUE_PREP` calls `workqueue_prepare_cpu` adds a worker to
      the per-cpu worker pool
    - `CPUHP_HRTIMERS_PREPARE` calls `hrtimers_prepare_cpu`
    - `CPUHP_SMPCFD_PREPARE`calls `smpcfd_prepare_cpu`
    - `CPUHP_RELAY_PREPARE`calls `relay_prepare_cpu`
    - `CPUHP_RCUTREE_PREP`calls `rcutree_prepare_cpu`
    - `CPUHP_TIMERS_PREPARE`calls `timers_prepare_cpu`
    - on x86, `CPUHP_BP_KICK_AP` calls `cpuhp_kick_ap_alive`
      - `cpuhp_can_boot_ap` updates per-cpu `cpuhp_state.ap_sync_state` to
        `SYNC_STATE_KICKED`
      - `native_kick_ap` kicks the secondary cpu
        - the secondary cpu is set up to call `start_secondary`
    - on x86 `CPUHP_BRINGUP_CPU` calls `cpuhp_bringup_ap`
      - `cpuhp_bp_sync_alive` waits for the cpu to become `SYNC_STATE_ALIVE`
        and updates the state to `SYNC_STATE_SHOULD_ONLINE`
      - `bringup_wait_for_ap_online`
        - `wait_for_ap_thread` waits for the secondary cpu to call
          `cpuhp_online_idle`
        - `kthread_unpark` starts secondary cpu's `cpuhp/%u`
          - the scheduler has been functioning since `sched_cpu_starting`
      - `cpuhp_kick_ap` wakes up `cpuhp/%u` thread and waits for it
    - on arm `CPUHP_BRINGUP_CPU` calls `bringup_cpu`
- `cpuhp_thread_fun` is per-cpu `cpuhp/%u` thread
  - `cpuhp_next_state` picks the next state based on `st->state` and
    `st->target`
    - it increments `st->state`
  - `cpuhp_invoke_callback` invokes the callbacks
  - `CPUHP_AP_SMPBOOT_THREADS` calls `smpboot_unpark_threads`
  - `CPUHP_AP_IRQ_AFFINITY_ONLINE` calls `irq_affinity_online_cpu`
  - `CPUHP_AP_PERF_ONLINE` calls `perf_event_init_cpu`
  - `CPUHP_AP_WATCHDOG_ONLINE` calls `lockup_detector_online_cpu`
  - `CPUHP_AP_WORKQUEUE_ONLINE` calls `workqueue_online_cpu`
  - `CPUHP_AP_RANDOM_ONLINE` calls `random_online_cpu`
  - `CPUHP_AP_RCUTREE_ONLINE` calls `rcutree_online_cpu`
  - `CPUHP_AP_ACTIVE` calls `sched_cpu_activate`
    - this activates the cpu for task migration and load balancing
    - scheduler has been functioning for per-cpu `swapper/%u` and
      `cpuhp/%u` since `sched_cpu_starting`
  - `CPUHP_ONLINE`
- on x86, secondary cpu jumps to `start_secondary` from `secondary_startup_64`
  - `cpuhp_ap_sync_alive` updates per-cpu `cpuhp_state.ap_sync_state` to
    `SYNC_STATE_ALIVE` and waits until it becomes
    `SYNC_STATE_SHOULD_ONLINE`
  - `ap_starting` calls `notify_cpu_starting` to bring the cpu up to
    `CPUHP_AP_ONLINE`
    - `CPUHP_AP_SCHED_STARTING` calls `sched_cpu_starting`
    - `CPUHP_AP_HRTIMERS_DYING` calls `hrtimers_cpu_starting`
    - `CPUHP_AP_ONLINE`
  - `cpu_startup_entry` calls `cpuhp_online_idle`
    - `cpuhp_ap_update_sync_state` updates to `SYNC_STATE_ONLINE`
    - `stop_machine_unpark` starts `migration/%u` thread
    - it updates `st->state` to `CPUHP_AP_ONLINE_IDLE`
    - `complete_ap_thread` notifies the boot cpu
