Kernel Clock Source
===================

## Basics

- a `clock_event_device` is a timer
  - `CLOCK_EVT_FEAT_PERIODIC`
  - `CLOCK_EVT_FEAT_ONESHOT`
- a `clocksource` is a free-running counter
  - it increments at a fixed freq
- the device tree defines timer nodes such as `arm,armv8-timer`
- `TIMER_OF_DECLARE` defines a driver to match timer nodes
  - it defines `__of_table_##name` in `__timer_of_table` section
  - `vmlinux.lds.h` has `TIMER_OF_TABLES` to collect them to
    `__timer_of_table` array
    - the last element is `__timer_of_table_sentinel`
- `timer_probe` probes the timers
  - e.g., `arch_timer_of_init` probes `arm,armv8-timer` node
  - `clocksource_register_hz` registers a `clocksource`
  - `clockevents_config_and_register` registers a `clock_event_device`
