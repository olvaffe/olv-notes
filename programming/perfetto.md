Perfetto
========

## Overview

- processes
  - there is a tracing service, `traced`
  - there are producers that connect to `traced`
    - `traced_probes`
    - every app that uses the tracing SDK
  - there is a consumer who can config, start, and stop tracing sessions
- tracing data overflow
  - each producer has a shared memory with `traced`
  - a producer can have several data sources
    - `traced_probes` has ftrace, `/proc/stat`, etc. as its data sources
    - different components of an app can be difference data sources
  - each data source writes `TracePacket` protos to the shared memory
    - the raw data are minimally parsed into protos
  - the consumer has one (or more) shared memory with `traced`
    - `traced` moves and mixes `TracePacket` protos from producer shared
      memories to the consumer shared memories
  - the consumer saves `TracePaket` protos to file
- analysis of a trace file
  - a `TraceProcessor` is a library that takes a trace file as an input,
    parses it into a in-memory SQL db, and provides SQL API to its clients
  - `TraceProcessor` supports native perfetto trace file, or other legacy
    trace files
    - android systrace
    - chrome json

## Running

- Build
  - `tools/install-build-deps`
    - `BUILD_DEPS_TOOLCHAIN_HOST`
      - `gn`
      - `clang-format`
      - `ninja`
      - `clang`
    - `BUILD_DEPS_HOST`
      - `googletest`
      - `protobuf`
      - `libcxx`, `libcxxabi`
      - `libunwind`
      - `fuzzer`
      - `benchmark`
      - `libbacktrace`
      - `sqlite`
      - `jsoncpp`
      - android core, unwind, logging, base, procinfo, bionic, etc.
      - `lzma`
      - `zlib`
      - `linenoise`
  - `tools/build_all_configs.py`
    - `LINUX_BUILD_CONFIGS`
      - `linux_gcc_debug`, `linux_gcc_release`
      - `linux_clang_debug`, `linux_clang_release`
      - various sanitizer builds
  - `tools/ninja -C out/linux_clang_release traced traced_probes perfetto`
- Run
  - start `traced` as a regular user
  - start `traced_probes` as root
    - or, make `/sys/kernel/debug` accessible by user
    - and `echo 0 > /sys/kernel/debug/tracing/tracing_on`
  - use `perfetto` to start/stop tracing
    - `perfetto -c configs/scheduling.cfg --txt --out aaa`

## Data Sources and Configs

- `data_sources { config { name: <...> <...>_config { ... }}}`
- most data sources are provided by `traced_probes`
  - `find -name '*_data_source.h'`
  - `InodeFileDataSource`
    - name: `linux.inode_file_map`
    - for io tracing
    - `inode_file_config`
  - `LinuxPowerSysfsDataSource`
    - name: `linux.sysfs_power`
    - probe `/sys/class/power_supply` for battery status
  - `ProcessStatsDataSource`
    - name: `linux.process_stats`
    - probe `/proc/<pid>` for process stats
      - also get process full names and which threads belong to which
        processes
    - `process_stats_config`
      - `proc_stats_poll_ms`
  - `SysStatsDataSource`
    - name: `linux.sys_stats`
    - probe `/proc/meminfo`, `/proc/vmstat`, `/proc/stat`,
      `/sys/class/devfreq`, `/sys/devices/system/cpu`, etc.
    - `sys_stats_config`
      - `meminfo_period_ms
      - `vmstat_period_ms`
      - `stat_period_ms`
      - `devfreq_period_ms`
      - `cpufreq_period_ms`
  - `SystemInfoDataSource`
    - name: `linux.system_info`
    - probe `/proc/cpuinfo`
  - `FtraceDataSource`
    - name: `linux.ftrace`
    - probe any ftrace event
    - `ftrace_config`
      - `ftrace_events`
  - `MetatraceDataSource`
    - name: `perfetto.metatrace`
    - to trace perfetto itself
  - `AndroidLogDataSource`
    - name: `android.log`
    - The "android.log" data source records log events from the Android log
      daemon (`logd`). These are the same log messages that are available via
      `adb logcat`.
    - `android_log_config`
  - `AndroidPowerDataSource`
    - name: `android.power`
    - This data source has been introduced in Android 10 (Q) and requires the
      presence of power-management hardware on the device. This is available
      on most Google Pixel smartphones.
    - `android_power_config`
  - `InitialDisplayStateDataSource`
    - name: `android.polled_state`
    - read the display state at the start of a trace (in case it does not
      change during the trace) and poll the display state periodically
    - `android_polled_state_config`
  - `PackagesListDataSource`
    - name: `android.packages_list`
    - read the Android package list for later deobfuscation
  - builtin data sources
    - `linux.perf` and `perf_event_config`
- a significant exception is `TrackEventDataSource`
  - name: `track_event`
  - `track_event_config`
    - `disabled_categories`
    - `enabled_categories`
    - `disabled_tags`
    - `enabled_tags`
- external data sources
  - `gpu.counters.*` and `gpu_counter_config`
  - `android.heapprofd` and `heapprofd_config`
  - `android.java_hprof` and `java_hprof_config`
  - `vulkan.memory_tracker` and `vulkan_memory_config`
  - `org.chromium.*` and `chrome_config`

## ftrace events

- `tools/ftrace_proto_gen/event_list`
  - process
    - `sched`
    - `task`
  - ipc
    - `binder`
    - `fastrpc`
    - `signal`
  - system
    - `cgroup`
    - `ftrace`
    - `kvm`
    - `synthetic`
    - `systrace`
    - `raw_syscalls`
    - `workqueue`
  - power
    - `power`
    - `regulator`
    - `thermal`
  - cpu
    - `cpuhp`
    - `ipi`
    - `irq`
  - mem
    - `compaction`
    - `kmem`
    - `lowmemorykiller`
    - `mm_event`
    - `oom`
    - `vmscan`
  - io
    - `block`
    - `ext4`
    - `f2fs`
    - `filemap`
    - `ufs`
  - net
    - `net`
    - `skb`
    - `sock`
    - `tcp`
  - gpu
    - `dmabuf_heap`
    - `dpu`
    - `fence`
    - `g2d`
    - `gpu_mem`
    - `ion`
    - `mali`
    - `mdss`
    - `sde`
    - `sync`
  - misc
    - `clk`
    - `cros_ec`
    - `i2c`
    - `scm`

## Use

- `PERFETTO_DEFINE_CATEGORIES(perfetto::Category("blah"))`
  - this defines a constexpr `kCategories` array
  - and declares a `g_category_state_storage` `std::atomic<uint8_t>` array
  - and defines `kConstExprCategoryRegistry` to wrap above
  - defines a new struct type, `TrackEvent` inheriting `DataSource`
- `PERFETTO_TRACK_EVENT_STATIC_STORAGE()`
  - definitions of declarations in `PERFETTO_DEFINE_CATEGORIES`
- `perfetto::Tracing::Initialize`
  - calls `Tracing::InitializeInternal`
- `perfetto::TrackEvent::Register`
  - builds `DataSourceDescriptor` proto
  - calls `TracingMuxerImpl::RegisterDataSource`
- `TRACE_EVENT_BEGIN("blah", "SliceName")`
  - `IsDynamicCategory` is false when used this way
  - compute the index of "blah" at compile-time
  - calls `TrackEventCategoryRegistry::GetCategoryState` to find out whether
    the category is enabled or not
  - if enabled, call `TrackEventDataSource::TraceForCategory` and write a
    `TracePacket`
- `TRACE_EVENT_END("blah")`
  - takes the same path as `TRACE_EVENT_BEGIN`, but with
    `TrackEvent::TYPE_SLICE_END` instead of `TrackEvent::TYPE_SLICE_BEGIN`
- the app is a producer and receives commands from the service to
  setup/start/stop data sources
  - `OnSetup`/`OnStart`/`OnStop` of `TrackEventDataSource` are called from
    `TracingMuxerImpl`
  - they will call `TrackEventInternal::EnableTracing` or
    `TrackEventInternal::DisableTracing`

## `TracingMuxerImpl`

- in-process backend
  - the process runs the service and consumers as well
  - used to collect trace events just for the app (e.g., chrome does this)
- system backend
 - the process is just a producer
 - used to collect trace events in the system-wide profiling
- `TracingMuxerImpl`
  - `Initialize`, for each backend, calls `ConnectProducer` (e.g.,
    `SystemTracingBackend::ConnectProducer`) to create an endpoint and wraps
    it in a `TracingMuxerImpl::ProducerImpl` 
  - for system backend, this connects to the system-wide producer socket.
  - `ProducerIPCClientImpl::OnServiceRequest` is called for requests from the
    service.  It calls back to `TracingMuxerImpl::ProducerImpl` for things
    such as `SetupDataSource`, `StartDataSource`, or `StopDataSource`
