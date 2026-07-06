# Android `atrace`

## Apps

- java apps and runtimes use `android.os.Trace` for tracing
  - apps always use `TRACE_TAG_APP`
  - runtimes can use any tag
- jni uses `libtracing_perfetto` for tracing
  - it can use either atrace or perfetto for tracing
  - `shouldPreferAtrace`
    - it returns false if atrace is disabled
    - it returns true if perfetto is diabled
      - java `Trace.registerWithPerfetto` must be called to enable perfetto
      - only `system_server`, sysui, and selected apps on
        `PERFETTO_TRACING_ALLOWLIST` call `Trace.registerWithPerfetto`
    - else, `debug.atrace.prefer_sdk` decides
  - if perfetto, it uses sdk to register matching categories and to trace
  - if atrace, it uses `cutils/trace.h`
- native apps use perfetto or atrace directly for tracing
  - ndk provides `ATrace_*` which always uses `ATRACE_TAG_APP`
  - platform code can use any tag

## Tracing Internals

- `atrace_init` is called on demand on hot paths
  - e.g., `atrace_get_enabled_tags` calls `atrace_init` and returns enabled
    tags
  - `atrace_seq_number_changed` is called on first time or
    `debug.atrace.tags.enableflags` has changed
    - if first time,
      - `atrace_property_info` is initialized
      - `atrace_init_once` performs one-time init
    - `atrace_update_tags` updates `atrace_enabled_tags` from
      `debug.atrace.tags.enableflags` and `debug.atrace.app_%d`
      - most tags are derived from `debug.atrace.tags.enableflags`
      - `ATRACE_TAG_APP` is from cmdline glob matching against
        `debug.atrace.app_%d`
- `atrace_int` traces a 32-bit counter
  - `atrace_is_tag_enabled` checks if the tag is enabled
  - `atrace_int_body` writes formated string to
    `/sys/kernel/tracing/trace_marker`

## Perfetto and atrace

- perfetto invokes `atrace --async_start --only_userspace <atrace_categories> -a <atrace_apps>`
  - the cli can enable/disable/dump tracing, but we don't use those
    functionalities
  - `g_categoryEnables` is from `<atrace_categories>`
  - `g_debugAppCmdLine` is from `<atrace_apps>`
  - `setUpUserspaceTracing`
    - `setAppCmdlineProperty` sets `debug.atrace.app_%d` and
      `debug.atrace.app_number` if `<atrace_apps>`
    - `setTagsProperty` sets `debug.atrace.tags.enableflags`
- perfetto enables `ftrace/print` implicitly when `atrace_categories` or
  `atrace_apps` are specified
  - atrace writes formatted strings to `/sys/kernel/tracing/trace_marker`
  - they show up in `/sys/kernel/tracing/per_cpu/cpuN/trace_pipe_raw` whether
    `ftrace/print` is enabled or not
  - but to tell perfetto to parse them, `ftrace/print` must be enabled
