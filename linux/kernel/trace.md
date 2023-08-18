Kernel trace
============

## `/sys/kernel/tracing`

- `core_initcall(tracefs_init)` creates `/sys/kernel/tracing` and registers
  `trace_fs_type`
- `fs_initcall(tracer_init_tracefs)` populates tracefs
  - `event_trace_init` adds `available_events` to tracefs
  - `init_tracer_tracefs` adds a bunch of files to tracefs
    - `available_tracers`
    - `current_tracer`
    - many more
  - `create_trace_instances` adds `instances`
