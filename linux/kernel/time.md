Kernel Time
===========

## Overview

- low-level mechanisms
  - `clockevents.c` provides `clock_event_device`, a timer
    - x86 mainly provides per-cpu `lapic` and global `hpet`
    - arm mainly provides per-cpu `arch_sys_timer` and emulated global `bc_hrtimer`
  - `clocksource.c` provides `clocksource`, a clock
    - x86 mainly provides `tsc`, `hpet`, and `acpi_pm`
    - arm mainly provides `arch_sys_counter`
  - `sched_clock.c` provides a fast path to read the best clock, bypassing
    `clocksource` when `CONFIG_GENERIC_SCHED_CLOCK` is enabled
    - x86 provides `sched_clock` directly instead to read tsc
    - arm calls `sched_clock_register` to register `arch_counter_get_cntvct`
- tick and hrtimer
  - `hrtimer.c` provides high-res timer to kernel and to userspace
    - it is based on `clock_event_device`
    - it calls `tick_program_event` to program `clock_event_device` indirectly
    - on irq, `hrtimer_interrupt` fires expired hrtimers
  - tick can be based on `clock_event_device` or hrtimer
    - during early boot, `tick_handle_periodic` is still the irq handler
      - `tick_periodic -> update_process_times -> run_local_timers ->
        hrtimer_run_queues -> hrtimer_switch_to_hres`
      - `tick_init_highres` switches to `hrtimer_interrupt` as the irq handler
      - `tick_setup_sched_timer` sets up `tick_nohz_handler` as a hrtimer
    - past early boot, when `hrtimer_interrupt` fires `tick_nohz_handler`,
      - `tick_sched_do_timer` calls `tick_do_update_jiffies64` to update jiffies
      - `tick_sched_handle` calls `update_process_times`
- tick-based mechanisms
  - `timer.c` provides tick-based timer to kernel
    - it always fires at jiffy boundary
  - `timekeeping.c` provides clocks to kernel and to userspace
    - it inits `CLOCK_REALTIME` using rtc
      - on x86, `read_persistent_clock64` returns cmos time
      - with `CONFIG_RTC_HCTOSYS`, `rtc_hctosys` also sets the time
    - it advances the clocks when tick calls `update_wall_time`
    - it uses `clocksource`s to extrapolate the current time
      - current time is "time at last tick" plus "hw cycles since last tick"
    - it supports `adjtimex`, which is how userspace ntp daemon adjusts time

## `clocksource`

- a `clocksource` is a counter that increments at a constant rate
- the main consumer is `timekeeping.c`
  - `tk_clock_read` reads the clock cycles
  - `timekeeping_cycles_to_ns` converts cycles to ns
- examples of `clocksource` on x86 in boot order
  - `refined_jiffies`, rating 2
  - `clocksource_hpet`, rating 250
  - `clocksource_tsc_early`, rating 299
  - `clocksource_jiffies`, rating 1
  - `clocksource_acpi_pm`, rating 200
  - `clocksource_tsc`, rating 300, `CLOCK_SOURCE_VALID_FOR_HRES`
- on arm, `clocksource` is mostly provided by `drivers/clocksource` and is
  typically `arch_sys_counter`
- `clocksource_register_khz` registers a khz clocksource
  - `scale * freq` is the frequency of counter increments per second
  - `__clocksource_update_freq_scale` initializes `mult` and `shift` from
    `scale` and `freq`
- `fs_initcall(clocksource_done_booting)`
  - until this point, clocksources are registered but unused
  - `clocksource_select`
    - `clocksource_find_best` finds the best clocksource
    - `timekeeping_notify` updates timekeeping clocksource

## `sched_clock`

- `sched_clock` is a function that returns nanoseconds since boot
  - it is used by scheduler, printk timestamp, etc.
  - it is like `clocksource`, but is cheaper at the cost of accuracy
- on x86, the arch provides `sched_clock` using tsc
- on arm, `CONFIG_GENERIC_SCHED_CLOCK` provides `sched_clock`
  - `sched_clock_register` registers a read callback for `sched_clock`
    - the read callback is typically `arch_counter_get_cntvct`
  - if no better read callback is registered, it defaults to the jiffy-based
    `jiffy_sched_clock_read`

## `clock_event_device`

- a `clock_event_device` is a device that can deliver an interrupt in the
  specified future periodically or one-shot
  - periodic is worse because it is less flexible and the error accumulates
- the main consumer is the tick subsystem or hrtimer
- examples of `clock_event_device` on x86 in boot order
  - `hpet_legacy_clockevent_register`, rating 50
    - because of `HPET_ID_LEGSUP` (legacy upgrade), it replaces
      `i8253_clockevent`
    - `timer_interrupt` is the irq handler
    - because of `X86_FEATURE_ARAT`, there is no per-cpu hpet, rating 110
      - which would use `hpet_msi_interrupt_handler` as the irq handler
  - `lapic_clockevent`, rating 150
    - because of `X86_FEATURE_ARAT`, it has rating 150 instead of 100
    - `local_apic_timer_interrupt` is the irq handler
- on arm, there are
  - `arch_timer_evt`, rating 450
    - `arch_timer_handler_virt` is the irq handler
  - `ce_broadcast_hrtimer`, rating 0
    - emulated broadcast timer based on hrtimer
    - `broadcast_needs_cpu` keeps one cpu timer on to drive hrtimer
- `clockevents_config_device` updates the struct
- `clockevents_register_device` adds the struct to `clockevent_devices` list
  - `tick_check_new_device` adds the device as the per-cpu or broadcast tick
    device
- `clockevents_config_and_register` is a shortcut to config and register
- `clockevents_switch_state` changes the state of the device
  - `CLOCK_EVT_STATE_PERIODIC` calls `set_state_periodic`
  - `CLOCK_EVT_STATE_ONESHOT` calls `set_state_oneshot`
- whenever an interrupt is received, the device driver calls `event_handler`
  to feed the interrupt into kernel tick subsystem

## Tick

- the tick subsystem uses `clock_event_device`s
  - a `clock_event_device` is a device that can fire time-based interrupts
  - a `tick_device` is a wrapper to `clock_event_device`
  - there are per-cpu `tick_cpu_device`
  - there is a global `tick_broadcast_device`
  - with `CONFIG_HIGH_RES_TIMERS`, it uses `clock_event_device` indirectly via
    hrtimer
- `tick_check_new_device` checks if a new clockevent dev is a better fit for
  the current cpu and switches to it for ticks
  - `tick_check_replacement`
    - clockevent dev can be cpu local or not, depending on its cpumask
    - non-local device could be treated as a local one if it can set irq
      affinity
    - if a device is cpu local, and better than the current one, we will use
      it as the new per-core clockevent dev
      - else, `tick_install_broadcast_device` checks if it is better than the
        current broadcast device
  - `tick_setup_device` sets up per-core clockevent dev
    - if it is the first device, mode is set to `TICKDEV_MODE_PERIODIC`
    - `tick_setup_periodic` sets the handler to `tick_handle_periodic`
      - it may switch the clockevent dev to `CLOCK_EVT_STATE_PERIODIC` or
        `CLOCK_EVT_STATE_ONESHOT` regardless of `TICKDEV_MODE_PERIODIC`,
        depending on what the clockevent dev is capable
        - it can emulate one state using another anyway
- `tick_handle_periodic` is called when the interrupt fires, and the mode is
  `TICKDEV_MODE_PERIODIC`
  - `tick_periodic` calls
    - `do_timer` increments `jiffies_64`
    - `update_wall_time` updates `timekeeper`
    - `update_process_times` charges 1 tick to the current process
      - `run_local_timers`
        - `hrtimer_run_queues` drives hrtimers
          - `tick_nohz_switch_to_nohz` potentially switch to low-res nohz mode
          - `hrtimer_switch_to_hres` potentially switch to high-res nohz mode
        - `raise_timer_softirq` raises `TIMER_SOFTIRQ` for wheel timers
      - `scheduler_tick` adds the tick to the scheduler
      - `run_posix_cpu_timers` checks if any userspace posix timer fires
  - `clockevents_program_event` arms the clockevent dev again if it is in
    `CLOCK_EVT_STATE_ONESHOT`
- `tick_nohz_handler` is called when the interrupt fires, and the mode is
  `TICKDEV_MODE_ONESHOT`
  - this happens after `hrtimer_switch_to_hres` switches to high-res nohz mode
    - if low-res nohz mode, `tick_nohz_lowres_handler` is used instead
  - actually, when the interrupt fires, `hrtimer_interrupt` handles the
    interrupt to drive hrtimers
    - `tick_nohz_handler` is merely an hrtimer
  - `tick_sched_do_timer`
    - `tick_do_update_jiffies64` advances `jiffies_64`
    - `update_wall_time` advances `timekeeper`
  - `tick_sched_handle`
    - `update_process_times` charges a tick to the current process
- broadcasting
  - if a per-core clockevent dev has `CLOCK_EVT_FEAT_C3STOP`, it sleeps when
    the core enters C3 sleep
  - the broadcast clockevent dev is responsible to wake up the core
  - this interacts with cpuidle
    - if cpuidle driver has any state with `CPUIDLE_FLAG_TIMER_STOP`,
      `tick_broadcast_enable` enables the broadcast clockevent dev
    - when `cpuidle_enter_state` enters a state with `CPUIDLE_FLAG_TIMER_STOP`,
      - `tick_broadcast_enter` stops the per-core dev and starts the broadcast dev
      - `tick_broadcast_exit` starts the per-core dev
  - `tick_handle_periodic_broadcast` is the handler when periodic
  - `tick_handle_oneshot_broadcast` is the handler when oneshot
  - `tick_do_broadcast` wakes up a sleeping core
    - x86 `lapic_timer_broadcast` sends `LOCAL_TIMER_VECTOR` ipi
    - arm `tick_broadcast` sends `IPI_TIMER` ipi
  - sleeping core wakes up to handle the ipi
    - x86 `local_apic_timer_interrupt`
    - arm `tick_receive_broadcast`
- there are 3 tick modes
  - `CONFIG_HZ_PERIODIC` ticks periodically at constant rate
  - `CONFIG_NO_HZ_IDLE` omits ticks on idle CPUs
  - `CONFIG_NO_HZ_FULL` omits ticks on CPUs that are idle or have one runnable
    task
  - unless `CONFIG_HZ_PERIODIC`, `tick_handle_periodic` is only used briefly
  - for `CONFIG_NO_HZ_IDLE` or `CONFIG_NO_HZ_FULL`, `tick_nohz_lowres_handler`
    or the high-resolution `hrtimer_interrupt` is used
    - `tick_sched_do_timer` updates `jiffies_64` and calls `update_wall_time`
    - `tick_sched_handle` calls `update_process_times`
- `NO_HZ` impl
  - both `NO_HZ` and `hrtimer` require oneshot mode
  - idle thread `do_idle`
    - `tick_nohz_idle_enter` sets `idle_entrytime` and marks
      `TS_FLAG_IDLE_ACTIVE` 
    - `tick_nohz_idle_stop_tick`
      - `tick_nohz_next_event` find the next event
      - `tick_nohz_stop_tick` stops tick until the next event
    - `tick_nohz_idle_exit` when no longer idle
      - `tick_nohz_idle_update_tick` calls `tick_nohz_restart_sched_tick` to
        start tick
  - `tick_handle_oneshot_broadcast` wakes up cpus that are in too deep idle
    states
    - `tick_do_broadcast` is called on cpus with expired `next_event`.  If it is
      cpu-of-execution, its `tick_device->evtdev->event_handler` is called for the
      rest, `tick_device->evtdev->broadcast` is called
    - If there is un-expired event, `tick_broadcast_set_event` is called to
      reprogram

## Timekeeping

- `timekeeping_init`
  - `read_persistent_wall_and_boot_offset` reads the wall time and the boot
    time from arch-specific clocks
    - on x86, wall time is from `mach_get_cmos_time` and boot time is from
      `native_sched_clock` which uses tsc
    - on arm64, wall time is dummy and boot time is from generic `sched_clock`
    - boot time is zeroed if it is greater than wall time
      - at this point, `wall_time + wall_to_mono = boot_offset`
  - `ntp_init` mainly provides `__do_adjtimex` which implements ntp time
    adjustment algorithm
  - `clocksource_default_clock` returns `clocksource_jiffies`
  - `tk_setup_internals` sets up the clocksource
  - `tk_set_xtime`, `tk_set_wall_to_mono` and `timekeeping_update_from_shadow`
    update for the first time since boot
- `update_wall_time` is called on ticks
  - `timekeeping_advance` reads the clocksource with `tk_clock_read`,
    calculates the delta since `cycle_last`, and applies the delta with
    `logarithmic_accumulation`
    - `logarithmic_accumulation` accumulates one cycle at a time and updates
      - `tk->tkr_mono.cycle_last`: to calculate delta next time
      - `tk->tkr_mono.xtime_nsec`: nanoseconds part of wall time
      - `tk->xtime_sec`: seconds part of wall time
      - and others
  - `timekeeping_update_from_shadow` calls `tk_update_ktime_data` to update
    fields used by ktime
    - `tk->tkr_mono.base`: current base monotonic ktime in nanoseconds
    - `tk->ktime_sec`: current monotonic ktime in seconds
    - and others
- `ktime_get` returns `CLOCK_MONOTONIC` in nanoseconds
  - `tk->tkr_mono.base` is in nanoseconds and was updated in the last tick
  - `timekeeping_get_ns` reads the clocksource and returns the delta since the
    last tick
  - summing them gives the current ktime in nanoseconds
- `clock_gettime(CLOCK_MONOTONIC)` calls `posix_get_monotonic_timespec` which
  calls `ktime_get_ts64`
  - `tk->xtime_sec` is the seconds part of the wall clock time updated in the
    last tick
  - `timekeeping_get_ns` returns the delta since the last tick
  - `tk->wall_to_monotonic` is the constant offset between wall time and
    monotonic time
  - summing them gives the current monotonic time
- `clock_gettime(CLOCK_REALTIME)` calls `posix_get_realtime_timespec` which
  calls `ktime_get_real_ts64`
  - this is similar to `ktime_get_ts64` but without `tk->wall_to_monotonic`

## NTP

- `timedatectl set-ntp yes` starts `systemd-timesyncd`
  - when synced, it calls `clock_adjtime(CLOCK_REALTIME)`
- `SYSCALL_DEFINE2(clock_adjtime, ...)`
  - `posix_clock_realtime_adj` calls `do_adjtimex`
  - `ntp_notify_cmos_timer` saves synced time to rtc
- `sync_hw_clock` is called on ntp sync or every `SYNC_PERIOD_NS` (11 minutes)
  - `update_persistent_clock64` is only for x86
    - `mach_set_cmos_time` calls `mc146818_set_time` directly
  - `update_rtc` updates `rtc0`

## hrtimer

- the functionality is always compiled
  - with `CONFIG_HIGH_RES_TIMERS`, high-res `clock_event_device` drives
    hrtimer, and hrtimer drives tick
  - without, `clock_event_device` drives tick, and tick drives hrtimer
- time-related posix syscalls make use of hrtimer
  - `timer_create -> common_timer_create -> hrtimer_setup`
  - `timer_gettime -> common_timer_get -> common_hrtimer_remaining`
  - `timer_settime -> common_timer_set -> common_hrtimer_arm`
  - `clock_nanosleep -> common_nsleep_timens -> hrtimer_nanosleep`
- `clock_event_device` irq handler
  - `tick_handle_periodic` is the initial handler
    - `-> tick_periodic -> update_process_times -> run_local_timers`
  - if nohz, `tick_nohz_switch_to_nohz` switches to `tick_nohz_lowres_handler`
    - `-> tick_nohz_handler -> tick_sched_handle -> update_process_times ...`
  - if `CONFIG_HIGH_RES_TIMERS`, `hrtimer_switch_to_hres` switches
    `hrtimer_interrupt`
    - `-> __hrtimer_run_queues -> __run_hrtimer -> tick_nohz_handler ...`
- <https://www.kernel.org/doc/html/latest/timers/highres.html>
  - `Hrtimers and Beyond`
  - Abstraction
    - Clock Source Management
      - abstract read-only clock sources
      - each clock souce expresses the time as a monotonically increasing value
      - helper functions exist to convert clock source's value into nanosecond
      - possible to list or choose from available sources
    - Clock Synchronization
      - reference time source (if available) is used to correct the
        monotonically increasing value
      - the value is read-only and thus the correction is a offset to it
    - Time-of-day Representation
    - Clock Event Management
      - currently, events defined to be periodic, with fixed period defined at
        compile time
      - which event sources to use is hardwired arch-specific
      - should abstract clock event sources
    - Removing Tick Dependencies
      - current timers assumed a fixed-period periodic event source
  - GTOD
    - Clock Source Management
    - Clock Synchronization
    - Time-of-day Representation
  - Clock Event Management
    - foundation for high resolution timers and dynamic tick
    - clock event source describes its properties, per-cpu or global
    - framework decides which function(s) are provided by which clock event
      source
    - periodic or oneshot modes.  And emulated periodic mode
    - event is distributed to periodic tick, process acct., profiling, and
      next event interrupt (e.g. hi-res timers and dyn tick)
- `hrtimer_switch_to_hres`
  - it is called to switch to high resolution mode when `hrtimer_run_queues`
    is called the first time
  - it calls `tick_init_highres`, `tick_setup_sched_timer`, and `retrigger_next_event`
  - if hres is disabled, `tick_check_oneshot_change` calls
    `tick_nohz_switch_to_nohz`
    - invokes `tick_switch_to_oneshot` with `tick_nohz_lowres_handler`
    - invokes `tick_setup_sched_timer` without `hrtimer`
- `tick_init_highres`
  - turns current cpu's `tick_device` to oneshot mode with `hrtimer_interrupt`
  - invokes `tick_broadcast_switch_to_oneshot` to turn broadcast event device
    to oneshot mode, with handler `tick_handle_oneshot_broadcast`
  - The next_event is set to `tick_next_period`
- `tick_setup_sched_timer`
  - install a hrtimer to emulate ticks
  - in its callback, `tick_sched_timer`, `tick_sched_do_timer` and
    `tick_sched_handle` are called and `HRTIMER_RESTART` is returned

## Boot (x86)

- `setup_kernel`
  - `setup_arch`
    - `x86_wallclock_init` disables `get_wallclock` and `set_wallclock` on
      special platforms
      - normally, they point to `mach_get_cmos_time` and `mach_set_cmos_time`
        to get/set the wallclock from/to CMOS
    - `register_refined_jiffies` to register (refined) jiffies as a clocksource
  - `tick_init`
    - `tick_broadcast_init`
    - `tick_nohz_init`
  - `init_IRQ`
  - `init_timers`
  - `hrtimers_init`
  - `timekeeping_init`
    - `ntp_init`
    - init current clocksource (`clocksource_jiffies`) for timekeeping purpose
    - xtime is initialized from `read_persistent_wall_and_boot_offset`
    - `wall_to_monotonic` initialized
      - `xtime + wall_to_monotonic == monotonic time`
  - `time_init`
  - `mem_init`
  - `late_time_init`
    - portion of `time_init` is delayed because we need `mem_init` for
      memory-mapped io
    - `x86_late_time_init`
      - `hpet_time_init`
        - `pit_timer_init` if could not `hpet_enable`
          - In that case, `clockevent_i8253_init` registers `i8253_clockevent`
            and `global_clock_event` is also set to `i8253_clockevent`
        - `setup_default_timer_irq` calls `request_irq(0, timer_interrupt, ...)`
      - `tsc_init` registers tsc clocksource
  - `sched_clock_init`
- `do_initcalls`
  - `init_jiffies_clocksource`
  - `clocksource_done_booting`
  - `hpet_late_init`
  - `init_tsc_clocksource`
