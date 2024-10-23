trace-cmd
=========

## Basic Usage

- `trace-cmd record -e drm -e dma_fence [cmd...]` traces `drm` and `dma_fence`
  events
  - it traces until cmd ends if a cmd is specified, or until ctrl-c
- `trace-cmd report` shows the events
- internally, `trace-cmd record`
  - inits ftrace
    - write 1 to `/sys/kernel/tracing/options/function-trace`
      - this option is ignored by `nop` tracer
    - write 1 to `/proc/sys/kernel/ftrace_enabled`
      - enables ftrace
    - write `nop` to `/sys/kernel/tracing/current_tracer`
      - sets current tracer to `nop`
    - write 0 to `/sys/kernel/tracing/events/enable`
      - disables all events
    - write 0 to `/sys/kernel/tracing/events/*/filter`
      - clears all filters
    - write ` ` to `/sys/kernel/tracing/set_ftrace_pid`
      - limits to pids
    - write 0 to `/sys/kernel/tracing/trace`
      - clears trace buffer
  - enables selected events and tracing
    - write 1 to `/sys/kernel/tracing/events/drm/enable`
      - enables `drm` events
    - write 1 to `/sys/kernel/tracing/events/dma_fence/enable`
      - enables `dma_fence` events
    - write 1 to `/sys/kernel/tracing/tracing_on`
      - enables the ring buffer
  - records events in children processes
    - fork a child per-cpu
    - read `/sys/kernel/tracing/per_cpu/cpuN/trace_pipe_raw` to
      `trace.dat.cpuN`
  - merges events
    - merge `trace.dat.cpuN` to `trace.dat`
  - inits ftrace again
- alternatively,
  - `trace-cmd start -e drm -e dma_fence` starts tracing
  - `trace-cmd show -p` shows events
  - `trace-cmd stop` stops tracing
  - or, `trace-cmd stream -e drm -e dma_fence` to start/show/stop

## Commands

- configuration
  - `set` configures ftrace
    - `-p` sets `/sys/kernel/tracing/current_tracer`
  - `reset` resets ftrace to default state
  - `stat` shows the current status
  - `options` is the same as `list -O`
  - `list` lists caps
    - `-e` lists events via `/sys/kernel/tracing/available_events`
    - `-s` lists event systems via `/sys/kernel/tracing/events`
    - `-t` lists tracers via `/sys/kernel/tracing/available_tracers`
    - `-o` lists options via `/sys/kernel/tracing/options`
    - `-f` lists filter functions via `/sys/kernel/tracing/available_filter_functions`
    - `-B` lists buffer instances via `/sys/kernel/tracing/instances`
    - `-C` lists clocks via `/sys/kernel/tracing/trace_clock`
    - `-P` lists plugins via `/usr/lib/traceevent/plugins`
    - `-O` lists plugin options, which are queried from plugins
    - `-c` lists compressions, which are determined at compile time
  - `check-events` discovers ill-formatted events
- tracing
  - `start` configures ftrace and starts tracing via `/sys/kernel/tracing/tracing_on`
  - `stop` stops tracing via `/sys/kernel/tracing/tracing_on`
  - `restart` starts tracing via `/sys/kernel/tracing/tracing_on` without re-configuration
  - `show` shows the contents of `/sys/kernel/debug/tracing/trace`
    - `-p` to show `/sys/kernel/debug/tracing/trace_pipe` instead
  - `clear` clears the ftrace ring buffer, by `echo 0 > /sys/kernel/tracing/trace`
  - `stream` is `start`, `show -p`, and `stop`
  - `profile` does ftrace-based profiling
  - `stack` enables stack tracer via `/proc/sys/kernel/stack_tracer_enabled`
  - `snapshot` takes/shows/resets a snapshot via `/sys/kernel/tracing/snapshot`
- recording
  - `record` configures ftrace, starts/stops tracing, and outputs `trace.dat`
    - on start, it forks per-cpu children to read `/sys/kernel/tracing/per_cpu/cpuN/trace_pipe_raw` to `trace.dat.cpuN`
    - on stop, it merges `trace.dat.cpuN` into `trace.dat`
  - `extract` read `/sys/kernel/tracing/per_cpu/cpuN/trace_pipe_raw` to `trace.data.cpuN` and merges them into `trace.data`
  - `restore` merges `trace.dat.cpuN` into `trace.data`, in case `record` crashes unexpectedly
- reporting
  - `report` shows `trace.data`
  - `hist` shows histogram of events
  - `split` splits `trace.data` into multiple smaller ones
  - `dump` shows metadata
  - `convert` converts between different trace file versions
- others
  - `listen` is used with `record -N`
  - `agent` is used with `record -A`
  - `setup-guest` sets up a guest fifo
  - `attach` combines a host trace and a guest trace
