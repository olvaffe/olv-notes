Kernel trace
============

## Initialization

- configs
  - `CONFIG_FTRACE` enables the tracing infra, which includes
    - `CONFIG_TRACEPOINTS`
    - `CONFIG_NOP_TRACER`
    - `CONFIG_EVENT_TRACING`
    - more
  - `CONFIG_FUNCTION_TRACER` enables the function tracer
    - it builds the kernel with `CC_FLAGS_FTRACE`
      - `-pg` inserts a call to `mcount` to all functions for tracing
  - `CONFIG_DYNAMIC_FTRACE` dynamically patches the call to `mcount`
  - `CONFIG_FTRACE_SYSCALLS` eanbles syscall tracing (unrelated to function
    tracer)
- `start_kernel` calls
  - `ftrace_init` patches `mcount` calls to nop
    - `ftrace_update_code` is called on each call and `ftrace_init_nop` does
      the patching
  - `early_trace_init`
    - `tracer_alloc_buffers`
      - it inits `global_trace`
      - it calls `register_tracer` to register `nop_trace`
      - `init_function_trace` registers `function_trace`
      - `init_events` calls `register_trace_event` to register trace events
        - these are not tracepoints
  - `trace_init` calls `trace_event_init`
    - `init_ftrace_syscalls` inits syscall tracing
    - `event_trace_enable` adds `trace_event_call` to `ftrace_events`
      - these are tracepoints
- `late_initcall_sync(late_trace_init)`
- `core_initcall(tracefs_init)`
  - `sysfs_create_mount_point` creates `/sys/kernel/tracing` as a mountpoint
- `fs_initcall(tracer_init_tracefs)`
  - `tracing_init_dentry` automounts legacy `/sys/kernel/debug/tracing`
  - `event_trace_init` adds `available_events`
  - `init_tracer_tracefs` adds a bunch of files
    - `available_tracers`
    - `current_tracer`
    - many more
  - `create_trace_instances` adds `instances`
    - `mkdir instances/foo` calls `trace_array_create`
      - this creates a new `trace_array` rather than using the `global_trace`
      - `tr->current_trace` is set to `nop_trace`
      - `allocate_trace_buffers` allocs the ring buffers
      - `trace_array_create_dir`
      - the trace array is added to `ftrace_trace_arrays`

## Tracepoints

- to define a tracepoint,
  - define `CREATE_TRACE_POINTS`
  - define `TRACE_SYSTEM`
  - include `linux/tracepoint.h`  
  - `DEFINE_EVENT`
    - this defines a `struct trace_event_call` in `_ftrace_events` section
  - include `trace/define_trace.h`
    - this defines a `struct tracepoint` in `__tracepoints` section
- when userspace writes 1 to `/sys/kernel/tracing/events/drm/enable`,
  - `system_enable_write` calls `ftrace_event_enable_disable` to enable the
    tracepoint
- when userspace writes 1 to `/sys/kernel/tracing/tracing_on`,
  - `rb_simple_write` calls `tracer_tracing_on` or `tracer_tracing_off`
  - `ring_buffer_record_on` enables the ring buffer
