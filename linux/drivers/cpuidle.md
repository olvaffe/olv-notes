Kernel cpuidle
==============

## Basics

- when there is no task to run, sched runs `do_idle`
- `do_idle` calls `cpuidle_idle_call` to enter the low-power idle states
  - `cpuidle_get_device` returns the `cpuidle_device`
  - `cpuidle_get_cpu_driver` returns the `cpuidle_driver`
  - `cpuidle_enter` enters the specified state
