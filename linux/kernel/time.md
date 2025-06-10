Kernel Time
===========

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
- on arm, `clocksource` is mostly provided by `drivers/clocksource`
- `clocksource_register_khz` registers a khz clocksource
  - `scale * freq` is the frequency of counter increments per second
  - `__clocksource_update_freq_scale` initializes `mult` and `shift` from
    `scale` and `freq`

## `sched_clock`

- `sched_clock` is a function that returns nanoseconds since boot
  - it is used by scheduler, printk timestamp, etc.
  - it is like `clocksource`, but is cheaper at the cost of accuracy
- on x86, the arch provides `sched_clock` using tsc
- on arm, `CONFIG_GENERIC_SCHED_CLOCK` provides `sched_clock`
  - `sched_clock_register` registers a read callback for `sched_clock`
  - if no better read callback is registered, it defaults to the jiffy-based
    `jiffy_sched_clock_read`

## `clock_event_device`

- a `clock_event_device` is a device that can deliver an interrupt in the
  specified future periodically or one-shot
- the main consumer is the tick subsystem
- x86 Hardware
  - CMOS Clock.  Could be programmed to alarm.  IRQ 8.
  - TSC: Time Stamp Counter, starting Pentium.  Same freq as the CPU.
  - PIT: Programmable Interval Timer.  Issue periodic irq on IRQ 0.  HZ ticks per second.
  - CPU local timer.  Part of local APIC.  Interupt only a local CPU.
  - HPET.  Registers are memory mapped.  Expect to replace PIT.  Interrupt ?.
  - ACPI PMT.
  - is clock event device per-cpu?  No spin lock needed when implementing one?
    - yes?
  - could a per-cpu clock event device also be a broadcast one at the same time?
    - no?
- examples of `clock_event_device` on x86
  - hpet
    - when `HPET_ID_LEGSUP` is set, hpet supports legacy replacement mode.
      Channel 0 is in `HPET_MODE_LEGACY` to replace legacy 8254 (and RTC).
      `timer_interrupt` calls its `event_handler`
    - when `X86_FEATURE_ARAT` is set, no channel is in `HPET_MODE_CLOCKEVT`
    - unused channels are put in `HPET_MODE_DEVICE`
  - lapic, for each core
  - `cat /proc/timer_list`
    - On my 6-core amd laptop, i have 7 clock event devices
    - a broadcast one, hpet, with `event_handler`
      `tick_handle_oneshot_broadcast`
    - two per-cpu ones, lapic, with `event_handler` `hrtimer_interrupt`
    - acpi is checked and TSC could be marked unstable
- `clockevents_config_device` updates the struct
- `clockevents_register_device` adds the struct to `clockevent_devices` list
  - `tick_check_new_device` adds the device as the per-cpu or broadcast tick
    device
- `clockevents_config_and_register` is a shortcut to config and register
- `clockevents_switch_state` changes the state of the device
  - `CLOCK_EVT_STATE_PERIODIC` calls `set_state_periodic`
  - `CLOCK_EVT_STATE_ONESHOT` calls `set_state_oneshot`
  - for hpet, `set_state_periodic` points to `hpet_clkevt_set_state_periodic`
    - hpet interrupts are handled by `timer_interrupt`
- whenever an interrupt is received, the device driver calls `event_handler`
  to feed the interrupt into kernel tick subsystem

## Tick

- the tick subsystem uses `clock_event_device`s
  - a `clock_event_device` is a device that can fire time-based interrupts
  - a `tick_device` is a wrapper to `clock_event_device`
  - there are per-cpu `tick_cpu_device`
  - there is a global `tick_broadcast_device`
- broadcasting
  - the tick subsystem prefers per-core `clock_event_device` 
  - if there is no per-core clockevent dev, the subsystem uses whatever
    clockevent dev as the broadcast device
  - `tick_do_broadcast` calls arch-specific `tick_broadcast`
    - on arm64, this sends `IPI_TIMER` to other cores
  - `tick_receive_broadcast` is called when a core receives the broadcast
    - on arm64, this is called when a core receives `IPI_TIMER`
- `tick_check_new_device` checks if a new clockevent dev is a better fit for
  the current cpu and switches to it for ticks
  - `tick_check_replacement`
    - clockevent dev can be cpu local or not, depending on its cpumask
    - non-local device could be treated as a local one if it can set irq
      affinity
    - if a device is cpu local, and better than the current one, goes on
  - if it is the first device, `tick_do_timer_cpu` is set to the cpu and the
    mode is set to `TICKDEV_MODE_PERIODIC`
  - `tick_setup_periodic` may switch the clockevent dev to
    `CLOCK_EVT_STATE_PERIODIC` or `CLOCK_EVT_STATE_ONESHOT` regardless of
    `TICKDEV_MODE_PERIODIC`, depending on what the clockevent dev is capable
    - it can emulate one state using another anyway
  - `clockevents_program_event` is called to arm the clockevent dev
- `tick_handle_periodic` is called when the interrupt fires, and the mode is
  `TICKDEV_MODE_PERIODIC`
  - `tick_periodic` calls
    - `do_timer` increments `jiffies_64`
    - `update_wall_time` updates `timekeeper`
    - `update_process_times` charges 1 tick to the current process
      - `run_local_timers` raises `TIMER_SOFTIRQ` and others
      - `scheduler_tick` adds the tick to the scheduler
      - `run_posix_cpu_timers` checks if any userspace posix timer fires
  - `clockevents_program_event` arms the clockevent dev again if it is in
    `CLOCK_EVT_STATE_ONESHOT`
- there are 3 tick modes
  - `CONFIG_HZ_PERIODIC` ticks periodically at constant rate
  - `CONFIG_NO_HZ_IDLE` omits ticks on idle CPUs
  - `CONFIG_NO_HZ_FULL` omits ticks on CPUs that are idle or have one runnable
    task
  - unless `CONFIG_HZ_PERIODIC`, `tick_handle_periodic` is only used briefly
  - for `CONFIG_NO_HZ_IDLE` or `CONFIG_NO_HZ_FULL`, `tick_nohz_handler` or the
    high-resolution `hrtimer_interrupt` is used
    - `tick_sched_do_timer` updates `jiffies_64` and calls `update_wall_time`
    - `tick_sched_handle` calls `update_process_times`
- in `NO_HZ`,
  - on any interrupt enter, `tick_irq_enter` is called to see if tick is
    disabled and needs to be re-enabled
    - `tick_nohz_update_jiffies` is called to catch up
  - on any interrupt exit, `tick_irq_exit` is called
    - `tick_nohz_irq_exit`

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

## hrtimer

- the functionality is always compiled
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
    - invokes `tick_switch_to_oneshot` with `event_handler`
      `tick_nohz_handler`
    - sets `nohz_mode` to `NOHZ_MODE_LOWRES`
    - initializes `sched_timer` without activating (it is used to record
      expiracy)
- `tick_init_highres`
  - turns current cpu's `tick_device` to oneshot mode with `event_handler`
    `hrtimer_interrupt`
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

## Old

- `NO_HZ`
  - periodic interrupt could be disabled when idle
  - catch up jiffies when the system becomes busy again
  - interrupts should also start again when the nearest timer expires
  - it is easier to switch to oneshot mode and program to fire when the
    nearest timer expires
  - and with a `event_handler` which programs itself to emulate periodic
    interrupt
- `hrtimer`
  - periodic interrupt no longer (accurate) enough
  - oneshot interrupt for the next_event is needed
  - a hrtimer which fires every jiffy is installed, which does what the old
    mechanism does
- IOW,
  - both `NO_HZ` and `hrtimer` require oneshot mode
  - `hrtimer` is more powerful than `NO_HZ`
- `kernel/time/timer.c` is low-resolution timer
- `kernel/time/hrtimer.c`
  - timers that usually do not expire (e.g., timers to handle io timeout) can
    keep using the low-resolution timer
  - hrtimer unit is nanosecond
  - timers are kept in a time-sorted, per-cpu, rb-tree
  - not super high resolution due to softirq
- Implementation
  - during boot time (until `device_initcall`), the clocksource used is always
    jiffies.
  - after then (after `clocksource_done_booting`), `clocksource_select` checks
    for a new timesource and switch to it
- `tick_nohz_stop_sched_tick`
  - invoked when cpu enters idle or `irq_exit`
  - invokes `tick_nohz_start_idle` to update the status of `tick_sched`
  - invokes `get_next_timer_interrupt` to get the jiffies of next timer or
    hrtimer event
  - if `next_jiffies` is more than one tick away, stop `tick_sched`
  - `ts->last_tick` is set to last tick so that we can catch up when recovered
  - start `sched_timer` in hires mode; otherwise, invokes `tick_program_event`
  - `clock_event_device` could have a `max_delta_ns` for merely several
    seconds.  timer may fire prematurely.
- `tick_nohz_idle_restart_tick`
  - invoked when cpu leaves idle
- `tick_handle_oneshot_broadcast`
  - `tick_do_broadcast` is called on cpus with expired `next_event`.  If it is
    cpu-of-execution, its `tick_device->evtdev->event_handler` is called for the
    rest, `tick_device->evtdev->broadcast` is called
  - If there is un-expired event, `tick_broadcast_set_event` is called to
    reprogram
