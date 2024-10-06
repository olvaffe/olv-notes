Kernel Clock Source
===================

## Basics

- a `clock_event_device` is a timer
  - `CLOCK_EVT_FEAT_PERIODIC`
  - `CLOCK_EVT_FEAT_ONESHOT`
- a `clocksource` is a free-running counter
  - it increments at a fixed freq
