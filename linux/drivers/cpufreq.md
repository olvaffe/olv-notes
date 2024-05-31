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

## `acpi-cpufreq`

- `acpi_scan_init` calls `acpi_processor_init`
  - when a processor is discovered, `acpi_processor_add` calls
    `acpi_processor_get_info`
  - if the processor object supports `_PCT` method (performance control),
    `cpufreq_add_device` is called to add `acpi-cpufreq` device
- `acpi_cpufreq_init` registers `acpi_cpufreq_platdrv`
  - this is called in `late_initcall` which happens later than amd-pstate
- `acpi_cpufreq_probe` probes the `acpi-cpufreq` dev
  - if there is a current driver, it bails
  - otherwise, it calls `cpufreq_register_driver` to register
    `acpi_cpufreq_driver`

## `amd_pstate`

- <https://docs.kernel.org/admin-guide/pm/amd-pstate.html>
- the AMD CPPC interface provides
  - read-only capabilities
    - Highest Performance
      - this is the absolute highest perf level
      - this is not sustainable
    - Nominal (Guaranteed) Performance
      - this is the highest sustainable perf level
    - Lowest non-linear Performance
      - this is the most efficient perf level
    - Lowest Performance
      - this is the absolute lowest perf level
- `amd_pstate_init`
  - it bails unless
    - the cpu vendor is `X86_VENDOR_AMD`
    - `acpi_cpc_valid` returns true, indicating `_CPC` (Continuous Performance
      Control) support
    - there is no cpufreq driver
  - `cppc_state` is `AMD_PSTATE_UNDEFINED` initially unless overridden by
    param
    - it checks acpi `FADT` table as well as cpu feature `X86_FEATURE_CPPC` to
      decide if `amd_pstate` should be used
    - it calls `amd_pstate_set_driver(CONFIG_X86_AMD_PSTATE_DEFAULT_MODE)`,
      which defaults to `AMD_PSTATE_ACTIVE` (3)
    - this picks `amd_pstate_epp_driver` (when active) or `amd_pstate_driver`
      (when passive or guided)
  - `amd_pstate_enable` enables cppc
  - `cpufreq_register_driver` registers the driver
    - note that this can fail, but since `amd_pstate_enable` is not undone,
      the cpu stays at the lowest perf level even when it falls back to
      `acpi-cpufreq`
