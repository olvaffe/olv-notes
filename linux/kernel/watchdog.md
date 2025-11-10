Kernel Watchdog
===============

## Overview

- `kernel_init_freeable` calls `lockup_detector_init` to init
  - init may be deferred and `late_initcall_sync(lockup_detector_check)`
    ensures init is completed
- `lockup_detector_reconfigure` configures watchdog
  - `softlockup_stop_all` stops watchdogs
    - it calls `watchdog_disable` on all cpus
  - `set_sample_period` updates `sample_period`
  - `lockup_detector_update_enable` updates `watchdog_enabled` flags
  - `softlockup_start_all` starts watchdogs
    - it calls `watchdog_enable` on all cpus
- `watchdog_enable` enables watchdog on a cpu
  - `softlockup_completion` is initialized and marked complete
  - `watchdog_hrtimer` is initialized and started
    - `watchdog_timer_fn` is the callback
    - `sample_period` is the period
  - `update_touch_ts`
    - `watchdog_touch_ts` is updated
    - `watchdog_report_ts` is updated
  - `watchdog_hardlockup_enable` enables hard watchdog
    - `watchdog_hardlockup_enable`
- `watchdog_disable` disables watchdog on a cpu
  - `watchdog_hardlockup_disable` disables hard watchdog
  - `hrtimer_cancel` cancels soft watchdog
- when `watchdog_timer_fn` runs,
  - `watchdog_hardlockup_kick` increments `hrtimer_interrupts`
  - `completion_done` typically returns true
    - `stop_one_cpu_nowait` stops the cpu (from other tasks) and calls
      `softlockup_fn` at the highest priority
  - `hrtimer_forward_now` schedules another run after `sample_period`
  - if `softlockup_fn` was not called within threshold, `is_softlockup`
    returns true
    - `BUG: soft lockup - CPU#%d stuck for %us! [%s:%d]`
- hardlock can have different meanings
  - `CONFIG_HARDLOCKUP_DETECTOR_PERF` uses perf nmi to call
    `watchdog_hardlockup_check` periodically
  - `CONFIG_HARDLOCKUP_DETECTOR_BUDDY` uses `watchdog_timer_fn` to call
    `watchdog_hardlockup_check` on the next cpu periodically
  - `is_hardlockup` returns true if `watchdog_timer_fn` hasn't called
    `watchdog_hardlockup_kick` to increment `hrtimer_interrupts`
    - `CPU%u: Watchdog detected hard LOCKUP on cpu %u`
- IOW,
  - when a cpu still handles watchdog hrtimer irq, but is stuck with the
    current task for `watchdog_thresh` (10) seconds, it is a soft lockup
  - when a cpu still handles perf nmi, but does not handle watchdog hrtimer
    irq for `watchdog_thresh` (10) seconds, it is a hard lockup
