Kernel Time
===========

## `clocksource`

- a `clocksource` is a counter that increments at a constant rate
- examples of `clocksource` on x86 in boot order
  - `refined_jiffies`, rating 2
  - `clocksource_hpet`, rating 250
  - `clocksource_tsc_early`, rating 299
  - `clocksource_jiffies`, rating 1
  - `clocksource_acpi_pm`, rating 200
  - `clocksource_tsc`, rating 300, `CLOCK_SOURCE_VALID_FOR_HRES`
- `clocksource_register_khz` registers a khz clocksource
  - `scale * freq` is the frequency of counter increments per second
  - `__clocksource_update_freq_scale` initializes `mult` and `shift` from
    `scale` and `freq`

## `clock_event_device`

- a `clock_event_device` is a device that can deliver an interrupt in the
  specified future periodically or one-shot
- examples of `clock_event_device` on x86
  - hpet
    - when `HPET_ID_LEGSUP` is set, hpet supports legacy replacement mode.
      Channel 0 is in `HPET_MODE_LEGACY` to replace legacy 8254 (and RTC).
      `timer_interrupt` calls its `event_handler`
    - when `X86_FEATURE_ARAT` is set, no channel is in `HPET_MODE_CLOCKEVT`
    - unused channels are put in `HPET_MODE_DEVICE`
  - lapic, for each core
- `clockevents_config_device` updates the struct
- `clockevents_register_device` adds the struct to `clockevent_devices` list
  - `tick_check_new_device` adds the device as the per-cpu or broadcast tick
    device
- `clockevents_config_and_register` is a shortcut to config and register
- whenever an interrupt is received, the device calls `event_handler` to feed
  the interrupt into kernel tick subsystem
  
## Tick

- the tick subsystem uses `clock_event_device`s
  - a `clock_event_device` is a device that can fire time-based interrupts
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
  - `clockevents_program_event` arms the clockevent dev again if it is in
    `CLOCK_EVT_STATE_ONESHOT`
- unless `CONFIG_HZ_PERIODIC`, `tick_handle_periodic` is only used briefly
- for `CONFIG_NO_HZ_IDLE` or `CONFIG_NO_HZ_FULL`, `tick_nohz_handler` or the
  high-resolution `hrtimer_interrupt` is used
  - `tick_sched_do_timer` updates `jiffies_64` and calls `update_wall_time`
  - `tick_sched_handle` calls `update_process_times`

## Timekeeping

- `update_wall_time`
- `ktime_get`
- `clock_gettime(CLOCK_REALTIME)` calls `posix_get_realtime_timespec` which
  calls `ktime_get_real_ts64`
- `clock_gettime(CLOCK_MONOTONIC)` calls `posix_get_monotonic_timespec` which
  calls `ktime_get_ts64`

## Boot (x86)

- `setup_kernel`
  - `setup_arch`
    - `x86_wallclock_init`
    - `register_refined_jiffies` to register (refined) jiffies as a clocksource
  - `tick_init`
    - `tick_broadcast_init`
    - `tick_nohz_init`
  - `init_timers`
  - `hrtimers_init`
  - `timekeeping_init`
  - `time_init`
  - `late_time_init`
    - `x86_late_time_init`
      - `hpet_time_init`
      - `tsc_init`
  - `sched_clock_init`
- `do_initcalls`
  - `init_jiffies_clocksource`
  - `clocksource_done_booting`
  - `hpet_late_init`
  - `init_tsc_clocksource`

## Wall clock

- `x86_wallclock_init` actually disables `get_wallclock` and `set_wallclock`
  on special platforms, which initially point to `mach_get_cmos_time` and
  `mach_set_rtc_mmss`
  - they read/set the wallclock from/to CMOS

## Scheduling-Clock Interrupts

- `CONFIG_HZ_PERIODIC` ticks periodically at constant rate
- `CONFIG_NO_HZ_IDLE` omits ticks on idle CPUs
- `CONFIG_NO_HZ_FULL` omits ticks on CPUs that are idle or have one runnable
  task
- a `tick_device` is a wrapper to `clock_event_device`
  - there are per-cpu `tick_cpu_device`
  - there is a global `tick_broadcast_device`
- `tick_setup_periodic` is called to set up periodic tick
  - `tick_set_periodic_handler` sets up the event handler
  - `__clockevents_switch_state` calls `set_state_periodic` callback to put
    the device in period mode
    - for hpet, it calls `hpet_clkevt_set_state_periodic`
    - the interrupts are handled by `timer_interrupt`
  - `tick_periodic` is eventually called from `tick_handle_periodic`
- `tick_periodic` is called in irq context
  - it increments `jiffies_64` by 1
  - it calls `update_process_times` and triggers timers
- when in `NO_HZ` mode, `tick_nohz_irq_enter` is called for any interrupt.
  - if tick was stopped, `tick_nohz_update_jiffies` is called to catch up

## Old

nohz:
- periodic interrupt could be disabled when idle
- catch up jiffies when the system becomes busy again
- interrupts should also start again when the nearest timer expires
- it is easier to switch to oneshot mode and program to fire when the nearest timer expires
- and with a event_handler which programs itself to emulate periodic interrupt

hires:
- periodic interrupt no longer (accurate) enough
- oneshot interrupt for the next_event is needed
- a hrtimer which fires every jiffy is installed, which does what the old mechanism does

summary:
- both nohz and hires requires oneshot mode
- hires is more powerful than nohz

## Timer Wheel

- `kernel/time/timer.c`
- CTW (cascade timer wheel)
- the unit is jiffies
- possible timer values in first category: 1, 2, 3, ..., 256 jiffies
- second category: 257, 513, 769, ..., 16384 jiffies
- on tick, timers might be moved, with IRQ disabled!
- moving timers from second category to the first might be expensive
- good performance, but has a bad worst case performance
- suit the need of networking and filesystem

## hrtimers

- those usually not expired (e.g., timer handle io timeout) could still use CTW
- the unit is nanosecond
- timers are kept in a time-sorted, per-cpu, rb-tree
- CLOCK_MONOTONIC and CLOCK_REALTIME
- not really high resolution due to softirq

Hrtimers and Beyond
http://tglx.de/projects/hrtimers/ols2006-hrtimers.pdf

GOAL: high-resolution timers, dynamic ticks
OLD: (HW | Clock source | TOD (time of day)) <-> timekeeping
OLD: (HW | Clock event source | ISR) -> tick

Abstraction:

- Clock Source Management
  - abstract read-only clock sources
  - each clock souce expresses the time as a monotonically increasing value
  - helper functions exist to convert clock source's value into nanosecond
  - possible to list or choose from available sources
- Clock Synchronization
  - reference time source (if available) is used to correct the monotonically increasing value
  - the value is read-only and thus the correction is a offset to it
- Time-of-day Representation
- Clock Event Management
  - currently, events defined to be periodic, with fixed period defined at compile time
  - which event sources to use is hardwired arch-specific
  - should abstract clock event sources
- Removing Tick Dependencies
  - current timers assumed a fixed-period periodic event source

GTOD:
- Clock Source Management
- Clock Synchronization
- Time-of-day Representation

Clock Event Management:
- foundation for high resolution timers and dynamic tick
- clock event source describes its properties, per-cpu or global
- framework decides which function(s) are provided by which clock event source
- periodic or oneshot modes.  And emulated periodic mode
- event is distributed to periodic tick, process acct., profiling, and next event interrupt (e.g. hi-res timers and dyn tick)

- Documentation/rtc.txt
- Documentation/timers/*

Hardware:
- CMOS Clock.  Could be programmed to alarm.  IRQ 8.
- TSC: Time Stamp Counter, starting Pentium.  Same freq as the CPU.
- PIT: Programmable Interval Timer.  Issue periodic irq on IRQ 0.  HZ ticks per second.
- CPU local timer.  Part of local APIC.  Interupt only a local CPU.
- HPET.  Registers are memory mapped.  Expect to replace PIT.  Interrupt ?.
- ACPI PMT.



- is clock event device per-cpu?  No spin lock needed when implementing one? yes?
- could a per-cpu clock event device also be a broadcast one at the same time? no?

Implementation:
- during boot time (until dev_initcalls), the clocksource used is always jiffies.
- after then (after clocksource_done_booting), update_wall_time checks for a new timesource and switch to it
- clock source registration enqueues the clock source only
- clock event device registration calls ->shutdown and ->set_mode
- 

my computer:
- On my 2core laptop, i have 3 clock event devices
- a broadcast one, hpet, with event_handler tick_handle_oneshot_broadcast
- two per-cpu ones, lapic, with event_handler hrtimer_interrupt
- cat /proc/timer_list
- acpi is checked and TSC could be marked unstable

On boot
- ... tick_init ... init_IRQ ... init_timers ... hrtimers_init ... timekeeping_init ... time_init ... mem_init ... late_time_init ...
- portion of time_init is delayed because we need mem_init for memory-mapped io

- there are {lapic,hpet,pit}_clockevent and more: They are timers for tick
- there are clocksource_{hpet,pit,tsc} and more: They are clocks for (high-accuracy) gettimeofday
- there is a default, not-yet-registered, clocksource_jiffies with the lowest rating.

timekeeping_init:
- invokes ntp_init
- init current clocksource (clocksource_jiffies) for timekeeping purpose
- xtime is initialized from read_persistent_clock.
- wall_to_monotonic initialized.  xtime + wall_to_monotonic == monotonic time

time_init:
- invokes tsc_init, which registers tsc clocksource.

late_time_init:
- usually points to hpet_time_init, which ...
- invokes setup_pit_timer() if could not hpet_enable().
- In that case, clockevents_register_device is invoked to register pit_clockevent.  global_clock_event also points to pit_clockevent.
- invokes time_init_hook() which invokes setup_irq(0, &irq0), which set up timer_interrupt as the handler.

tick_notify:
- tick_init calls clockevents_register_notifier to register tick_notify
- is invoked when, for example, pit_clockevent is registered.
- invokes tick_check_new_device when new device is added.

tick_check_new_device:
- devices are cpu local or not, depending on its cpumask
- non-local device could be treated as a local one if it can set irq affinity
- if a device is cpu local, and better than the current one, tick_setup_device and tick_oneshot_notify
- if no previous device on this cpu, and no tick_do_timer_cpu, it is used for tick_do_timer_cpu
- if device is DUMMY (like lapic ones), event_handler is set to tick_handle_periodic...
- and done.  it is shutdown and waiting for broadcast
- tick_oneshot_notify sets first bit of tick_sched->check_clocks
- for device that is not cpu local, tick_check_broadcast_device and tick_broadcast_start_periodic
- if it is better than current broadcast device, replace it

void tick_setup_periodic(struct clock_event_device *dev, int broadcast)
- dev->event_handler is set to either tick_handle_periodic or tick_handle_periodic_broadcast
- if dev is DUMMY, that's all
- if broadcast device is in oneshot mode, or dev does not support PERIODIC, it is put in oneshot mode
- with tick_next_period as next event
- otherwise, periodic

tick_handle_periodic:
- if current cpu is tick_do_timer_cpu, invokes do_timer() to update jiffies_64 and xtime
- invokes update_process_times() to, among others, raise_softirq() and scheduler_tick()

tick_handle_periodic_broadcast:
- if current cpu is in tick_broadcast_mask, remove from the mask and call evdev->event_handler
- then evdev->broadcast on tick_broadcast_mask
- periodically!

summary before hrtimers:
- at this stage, 2 lapic timer are shutdown with event_handler tick_handle_periodic
- hpet fires periodically with event_handler tick_handle_periodic_broadcast
- thus calling one of the lapic's event_handler

hrtimers:
- hrtimer_run_pending is called from timer's softirq and hrtimers is switched to hires mode through hrtimer_switch_to_hres, in which...
- calls tick_init_highres, tick_setup_sched_timer, and retrigger_next_event
- unless hires is disabled, then tick_nohz_switch_to_nohz

tick_nohz_switch_to_nohz:
- invokes tick_switch_to_oneshot with event_handler tick_nohz_handler
- sets nohz_mode to NOHZ_MODE_LOWRES
- updates last_jiffies_update
- initializes sched_timer without activating (it is used to record expiracy)

tick_nohz_stop_sched_tick:
- invoked when cpu enters idle or irq_exit
- invokes tick_nohz_start_idle to update the status of tick_sched
- invokes get_next_timer_interrupt to get the jiffies of next timer or hrtimer event
- if next_jiffies is more than one tick away, stop sched_tick
- current cpu is set in nohz_cpu_mask
- ts->idle_tick is set to last tick so that we can catch up when recovered
- start sched_timer in hires mode; otherwise, invokes tick_program_event
- clock_event_device could have a max_delta_ns for merely several seconds.  timer will fire prematurely.

tick_check_idle:
- invoked in irq_enter if cpu is idle

tick_nohz_restart_sched_tick:
- invoked when cpu leaves idle

tick_init_highres:
- turns current cpu's tick_device to oneshot mode with event_handler hrtimer_interrupt.
- invokes tick_broadcast_switch_to_oneshot to turn broadcast event device to oneshot mode, with handler tick_handle_oneshot_broadcast.
  The next_event is set to tick_next_period.

tick_setup_sched_timer:
- install a hrtimer to emulate ticks
- in its callback, tick_do_update_jiffies64, update_process_times and profile_tick are called, and it returns HRTIMER_RESTART.

tick_handle_oneshot_broadcast:
- tick_do_broadcast is called on cpus with expired next_event.  If it is cpu-of-execution, its tick_device->evtdev->event_handler is called
  for the rest, tick_device->evtdev->broadcast is called
- If there is un-expired event, tick_broadcast_set_event is called to reprogram
