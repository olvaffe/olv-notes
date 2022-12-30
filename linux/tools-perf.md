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

## `clock_nanosleep`

- an idle system and a program in a tight loop that sleeps for different times
- sleep 1us
  - top says 10.5% cpu time
  - perf record says 24.5% cpu cycles
  - perf stat says 20% cpu
- sleep 10us
  - top says 9% cpu time
  - perf record says 20.5% cpu cycles
  - perf stat says 15% cpu
- sleep 100us
  - top says 4% cpu time
  - perf record says 11% cpu cycles
  - perf stat says 7% cpu
- sleep 1ms
  - top says 1.5% cpu time
  - perf record says 7% cpu cycles
  - perf stat says 2% cpu
- sleep 10ms
  - top says 0.5% cpu time
  - perf record says 3.5% cpu cycles
  - perf stat says 0.3% cpu

## symbols

- `/proc/sys/kernel/kptr_restrict`
  - default 0 on arch linux
- `/proc/sys/kernel/perf_event_paranoid`
  - default 2 on arch linux
- just use sudo to record and report
- `perf record` has debuginfod integration
  - when `DEBUGINFOD_URLS` is set, it downloads all symbols and is slow at the
    end of the first run
