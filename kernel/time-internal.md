* is clock event device per-cpu?  No spin lock needed when implementing one? yes?
* could a per-cpu clock event device also be a broadcast one at the same time? no?

Implementation:
* during boot time (until dev_initcalls), the clocksource used is always jiffies.
* after then (after clocksource_done_booting), update_wall_time checks for a new timesource and switch to it
* clock source registration enqueues the clock source only
* clock event device registration calls ->shutdown and ->set_mode
* 

my computer:
* On my 2core laptop, i have 3 clock event devices
* a broadcast one, hpet, with event_handler tick_handle_oneshot_broadcast
* two per-cpu ones, lapic, with event_handler hrtimer_interrupt
* cat /proc/timer_list
* acpi is checked and TSC could be marked unstable

On boot
* ... tick_init ... init_IRQ ... init_timers ... hrtimers_init ... timekeeping_init ... time_init ... mem_init ... late_time_init ...
* portion of time_init is delayed because we need mem_init for memory-mapped io

* there are {lapic,hpet,pit}_clockevent and more: They are timers for tick
* there are clocksource_{hpet,pit,tsc} and more: They are clocks for (high-accuracy) gettimeofday
* there is a default, not-yet-registered, clocksource_jiffies with the lowest rating.

timekeeping_init:
* invokes ntp_init
* init current clocksource (clocksource_jiffies) for timekeeping purpose
* xtime is initialized from read_persistent_clock.
* wall_to_monotonic initialized.  xtime + wall_to_monotonic == monotonic time

time_init:
* invokes tsc_init, which registers tsc clocksource.

late_time_init:
* usually points to hpet_time_init, which ...
* invokes setup_pit_timer() if could not hpet_enable().
* In that case, clockevents_register_device is invoked to register pit_clockevent.  global_clock_event also points to pit_clockevent.
* invokes time_init_hook() which invokes setup_irq(0, &irq0), which set up timer_interrupt as the handler.

tick_notify:
* tick_init calls clockevents_register_notifier to register tick_notify
* is invoked when, for example, pit_clockevent is registered.
* invokes tick_check_new_device when new device is added.

tick_check_new_device:
* devices are cpu local or not, depending on its cpumask
* non-local device could be treated as a local one if it can set irq affinity
* if a device is cpu local, and better than the current one, tick_setup_device and tick_oneshot_notify
* if no previous device on this cpu, and no tick_do_timer_cpu, it is used for tick_do_timer_cpu
* if device is DUMMY (like lapic ones), event_handler is set to tick_handle_periodic...
* and done.  it is shutdown and waiting for broadcast
* tick_oneshot_notify sets first bit of tick_sched->check_clocks
* for device that is not cpu local, tick_check_broadcast_device and tick_broadcast_start_periodic
* if it is better than current broadcast device, replace it

void tick_setup_periodic(struct clock_event_device *dev, int broadcast)
* dev->event_handler is set to either tick_handle_periodic or tick_handle_periodic_broadcast
* if dev is DUMMY, that's all
* if broadcast device is in oneshot mode, or dev does not support PERIODIC, it is put in oneshot mode
* with tick_next_period as next event
* otherwise, periodic

tick_handle_periodic:
* if current cpu is tick_do_timer_cpu, invokes do_timer() to update jiffies_64 and xtime
* invokes update_process_times() to, among others, raise_softirq() and scheduler_tick()

tick_handle_periodic_broadcast:
* if current cpu is in tick_broadcast_mask, remove from the mask and call evdev->event_handler
* then evdev->broadcast on tick_broadcast_mask
* periodically!

summary before hrtimers:
* at this stage, 2 lapic timer are shutdown with event_handler tick_handle_periodic
* hpet fires periodically with event_handler tick_handle_periodic_broadcast
* thus calling one of the lapic's event_handler

hrtimers:
* hrtimer_run_pending is called from timer's softirq and hrtimers is switched to hires mode through hrtimer_switch_to_hres, in which...
* calls tick_init_highres, tick_setup_sched_timer, and retrigger_next_event
* unless hires is disabled, then tick_nohz_switch_to_nohz

tick_nohz_switch_to_nohz:
* invokes tick_switch_to_oneshot with event_handler tick_nohz_handler
* sets nohz_mode to NOHZ_MODE_LOWRES
* updates last_jiffies_update
* initializes sched_timer without activating (it is used to record expiracy)

tick_nohz_stop_sched_tick:
* invoked when cpu enters idle or irq_exit
* invokes tick_nohz_start_idle to update the status of tick_sched
* invokes get_next_timer_interrupt to get the jiffies of next timer or hrtimer event
* if next_jiffies is more than one tick away, stop sched_tick
* current cpu is set in nohz_cpu_mask
* ts->idle_tick is set to last tick so that we can catch up when recovered
* start sched_timer in hires mode; otherwise, invokes tick_program_event
* clock_event_device could have a max_delta_ns for merely several seconds.  timer will fire prematurely.

tick_check_idle:
* invoked in irq_enter if cpu is idle

tick_nohz_restart_sched_tick:
* invoked when cpu leaves idle

tick_init_highres:
* turns current cpu's tick_device to oneshot mode with event_handler hrtimer_interrupt.
* invokes tick_broadcast_switch_to_oneshot to turn broadcast event device to oneshot mode, with handler tick_handle_oneshot_broadcast.
  The next_event is set to tick_next_period.

tick_setup_sched_timer:
* install a hrtimer to emulate ticks
* in its callback, tick_do_update_jiffies64, update_process_times and profile_tick are called, and it returns HRTIMER_RESTART.

tick_handle_oneshot_broadcast:
* tick_do_broadcast is called on cpus with expired next_event.  If it is cpu-of-execution, its tick_device->evtdev->event_handler is called
  for the rest, tick_device->evtdev->broadcast is called
* If there is un-expired event, tick_broadcast_set_event is called to reprogram
