Linux perf
==========

## Event Counting

- test configuration
  - 3GHz CPU
  - compare idling (sleep 1) and busy looping for 1s
- HW
  - `perf stat -e branches <cmd>`
    - idling reports ~230K branches
    - busy looping reports ~465M branches
  - `perf stat -e branch-misses <cmd>`
    - idling reports ~7K branch misses
    - busy looping reports ~10K branch misses
  - `perf stat -e cache-misses <cmd>`
    - idling reports ~700
    - busy looping reports ~5K
  - `perf stat -e cycles <cmd>`
    - idling reports ~1M
    - busy looping reports ~3G
  - `perf stat -e instructions <cmd>`
    - idling reports ~764K
    - busy looping reports ~2.3G
- SW
  - `perf stat -e context-switches <cmd>`
    - idling reports ~1
    - busy looping reports ~2
  - `perf stat -e cpu-clock <cmd>`
    - idling reports ~1ms
    - busy looping reports ~1s
  - `perf stat -e page-faults <cmd>`
    - idling reports ~58
    - busy looping reports ~43
  - `perf stat -e task-clock <cmd>`
    - idling reports ~1ms
    - busy looping reports ~1s
- Tracepoint
  - `perf stat -e dma_fence:* <cmd>`
  - `perf stat -e drm:* <cmd>`
  - `perf stat -e kvm:* <cmd>`
  - `perf stat -e power:* <cmd>`
  - `perf stat -e sched:* <cmd>`
  - `perf stat -e syscalls:* <cmd>`

# Tracepoints

- `cd /sys/kernel/debug/tracing`
  - for help, `cat README`
  - clear trace: `echo > trace`
  - enable a tracepoint: `echo 1 > events/power/cpu_frequency/enable`
  - start tracing: `echo 1 > tracing_on`
  - stop tracing: `echo 0 > tracing_on`
  - get trace: `cat trace`
- list all events
  - `find /sys/kernel/debug/tracing/events -type d`
