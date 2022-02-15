Linux perf
==========

## Event Counting

- test configuration
  - `i7-7820HQ`
    - `cat /sys/devices/system/cpu/cpu0/cpufreq/base_frequency`
      - 2900000
    - `cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq`
      - `3900000`
  - compare idling and busy looping for 1s
    - `perf stat --timeout 1000 blah`
- HW events
  - branches: 211k vs. 3825M
  - branch-misses: 7k vs. 11k
  - cache-misses: 28k vs. 28k
  - cycles: 1M vs. 3834M
  - instructions: 1M vs. 3827M
- SW events
  - context-switches: 1 vs. 3
  - cpu-clock: 1.3ms vs. 1000ms
  - page-faults: 63 vs. 45
  - task-clock: 1.3ms vs. 1000ms
- Tracepoint
  - `perf stat -e dma_fence:* <cmd>`
  - `perf stat -e drm:* <cmd>`
  - `perf stat -e kvm:* <cmd>`
  - `perf stat -e power:* <cmd>`
  - `perf stat -e sched:* <cmd>`
  - `perf stat -e syscalls:* <cmd>`

## Event Sampling

- `perf record -e cycles -c 1000 <cmd>`
  - generate an interrupt and collect a sample for every 1000 cycles
  - every 1000000 cycles is more appropriate for cycles
- `perf record -e cycles -F 1000 <cmd>`
  - generate 1000 interrupts and colloect 1000 samples every second

## Tracepoints

- `cd /sys/kernel/debug/tracing`
  - for help, `cat README`
  - clear trace: `echo > trace`
  - enable a tracepoint: `echo 1 > events/power/cpu_frequency/enable`
  - start tracing: `echo 1 > tracing_on`
  - stop tracing: `echo 0 > tracing_on`
  - get trace: `cat trace`
- list all events
  - `find /sys/kernel/debug/tracing/events -type d`
