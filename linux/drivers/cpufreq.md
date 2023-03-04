Kernel cpufreq
==============

## cpufreq

- <https://docs.kernel.org/admin-guide/pm/cpufreq.html>
- three layers
  - the core provides the common code and the userspace interfaces
  - each scaling governor implements an algorithm to estimate required cpu
    capacity
  - each scaling driver talks to the hw
- in principle, any scaling governor/driver combo can work
  - however, some algorithms make use of hw-specific knowledge
  - they are implemented inside scaling drivers rather than as governors
  - `intel_pstate` is an example and it can bypass governors
- at anytime, there can only be a single scaling driver managing all cpus
- scaling governors
  - `performance` always requests the highest frequency, within the limit of
    `scaling_max_freq`
    - the request is made when `scaling_governor`, `scaling_max_freq`, or
      `scaling_min_freq` are changed
  - `powersave` always requests the lowest frequency, within the limit of
    `scaling_min_freq`
    - the request is made when `scaling_governor`, `scaling_max_freq`, or
      `scaling_min_freq` are changed
  - `userspace` requests the frequency specified by `scaling_setspeed`
  - `schedutil` makes use of cpu utilization data available from the scheduler
  - `ondemand` uses cpu active time over the last `sampling_rate`us to
    determine the cpu load
  - `conservative` is similar to `ondemand`, except it adjusts the frequencies
    in small steps
- `/sys/devices/system/cpu/cpufreq/policyX`
  - cur, max, min freq of a CPU
  - cur, max, min freq used by the scaling governor
  - current/available governor
  - benchmarks
    - `dd if=/dev/zero of=/dev/null bs=1M count=10000`
    - `sysbench --time=3 --threads=1 cpu run | grep total`
  - for example,
    - `cd /sys/devices/system/cpu`
    - `echo 0 | sudo tee cpu?/online` to bring all but cpu0 offline
    - `cat cpufreq/policy0/cpuinfo_min_freq | sudo tee cpufreq/policy0/scaling_max_freq`
    - this should make a big difference in benchmarks

## `intel_pstate`

- <https://www.kernel.org/doc/Documentation/cpu-freq/intel-pstate.txt>
- two modes
  - in the active mode, it bypasses the scaling governor
  - in the passive mode, it is like a regular scaling driver
- active mode
  - this is the default mode on newer cpus, where there are hardware-managed
    P-states (HWP) support
  - it bypasses the scaling governor and provides its own algorithms
    - confusingly, they are called `powersave` and `performance` but behave
      differently from the generic ones
  - if HWP is available and enabled,
    - `performance` forces EPP/EPB to 0, meaning the processor's internal
      p-state selection logic focuses on performance and also only selects
      higher p-states
    - `powersave` does not change EPP/EPB value.  If it is 1, it is more
      balanced
      - this is similar to the generic `schedutil` except this is implemented
        in hw
  - if HWP is unavailable or disabled,
    - `performance` always selects the highest allowed p-state
    - `powersave` is similar to the generic `schedutil`, except the
      utilization data is from hw rather than from the cpu scheduler
- passive mode
  - this is the default mode on older cpus, where there are no HWP
  - `scaling_driver` reports `intel_cpufreq` rather than `intel_pstate`
  - it can be paried with any generic scaling governors
- `/sys/devices/system/cpu/intel_pstate`
  - `num_pstates` is 32, meaning there are 32 states
  - `turbo_pct` is also 32, meaning there are `32%*num_pstates` in the turbo
    range
  - setting `no_turbo` to 1 makes pstates in the turbo change unavailable
  - `max_perf_pct` is 100 meaning all available pstates can be used
    - i.e., 32 states with turbo or 22 states without turbo
  - `min_perf_pct` is 20 meaning the minimum pstate is 20% of non-turbo
    states
- except `intel_pstate` does not work on an intel machine
  - changing `scaling_max_freq` has no effect
  - changing `max_perf_pct` has no effect
  - and it works now after a reboot....
