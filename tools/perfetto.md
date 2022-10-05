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

- <https://perfetto.dev/docs/contributing/build-instructions>
  - `git clone https://android.googlesource.com/platform/external/perfetto`
  - `tools/install-build-deps [--linux-arm]` to install
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
    - sysroot
  - `tools/gn args out/blah`
    - `target_os = "linux"`
    - `target_cpu = "x64" or "arm64"`
    - `is_debug = false`
    - `cc_wrapper = "ccache"`
  - `ninja -C out/blah`
  - `scp out/blah/tracebox dst:~`
- Run
  - start `tracebox traced` as a regular user
  - start `tracebox traced_probes --reset-ftrace` as root
    - or, make `/sys/kernel/debug` accessible by user
  - use `perfetto` to start/stop tracing
    - `perfetto -c configs/scheduling.cfg --txt --out aaa`
- to convert txt config to binary config,
  - install `protobuf-compiler` to get `protoc`
  - the config proto file is `protos/perfetto/config/perfetto_config.proto`
    - <https://raw.githubusercontent.com/google/perfetto/master/protos/perfetto/config/perfetto_config.proto>
  - `protoc --encode=perfetto.protos.TraceConfig perfetto_config.proto < config.txt > config.bin`
- Android
  - <https://perfetto.dev/docs/quickstart/android-tracing>
  - use <https://raw.githubusercontent.com/google/perfetto/master/tools/record_android_trace>
  - or, `cat config.txt | adb shell perfetto --txt -c - -o /data/misc/perfetto-traces/trace`
  - or, on older devices,
    - `adb shell setprop persist.traced.enable 1`
    - convert `config.txt` to `config.bin`
    - `cat config.bin | adb shell perfetto -c - -o /data/misc/perfetto-traces/trace`
    - categories to enable
      - `am binder_driver dalvik freq gfx hal idle input res view sched wm`

## Tools

- `traceconv` to convert protobuf binary to protobuf text, systrace, etc.
  - protoc can be used to decode binary to txt as well
    - `protoc --decode=perfetto.protos.Trace protos/perfetto/trace/perfetto_trace.proto`
  - protoc can support encode which `traceconv` can't
- `trace_processor_shell` is a sql shell
  - `.dump <file>` to save to sqlite
  - `select * from track` shows all tracks
  - `select * from thread_track` shows all tracks that are threads
    - perfetto uses an OO design
  - `select * from slice` shows all slices
- newbie way
  - `select utid from thread where name == blah` to find `utid` of `blah`
  - `select id from thread_track where utid == foo` to find the track id
  - `select ts, dur, name, depth from slice where track_id == bar order by ts`
    to find the slices

## Release Process

- `master` is the development branch
- `ui.perfetto.dev` uses `ui/release/channels.json`
  - user can pick from `stable`, `canary`, or `autopush` channels
  - `autopush` channel uses master tot
  - `canary` uses `ui-canary` branch
    - every 1-2 weeks, `master` is merged to `ui-canary` and the json file is
      updated
    - there can also be cherry-picks to fix issues
  - `stable` uses `ui-stable` branch
    - every 4 weeks, `ui-canary` is merged to `ui-stable` and the json file is
      updated
    - there can also be cherry-picks to fix issues
- SDK releases
  - once in a while, `releases/v<VER>.x` is branched from master
  - `tools/gen_amalgamated --output sdk/perfetto`
  - `git add sdk/perfetto.{cc,h}`
  - `git commit -m "Amalgamated source for vX.Y"`

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

## SQL

- <https://perfetto.dev/docs/analysis/sql-tables>
- object-oriented tables
- `track` table
  - columns: `id`, `type`, `name`, ...
  - `thread_track` table
    - additional columns: `utid`
  - `process_track` table
    - additional columns: `utid`
  - `gpu_track` table
    - additional columns: `context_id`, ...
  - `counter_track` table
  - ...
- `slice` table
  - columns: `id`, `type`, `ts`, `dur`, `name`, `track_id`, `arg_set_id`,
    `slice_id`, ...
- `thread` table
  - columns: `utid`, `id`, `type`, `tid`, `name`, `start_ts`, `end_ts`,
    `upid`, `is_main_thread`, ...
- `process` table
- `args` table
  - columns: `id`, `type`, `arg_set_id`, `flat_key`, `key`, `int_value`,
    `string_value`, ...

## protobuf

- implementations
  - the official one is <https://github.com/protocolbuffers/protobuf>
  - C library, <https://github.com/protobuf-c/protobuf-c>
  - minimal and high-performance, <https://github.com/mapbox/protozero>
  - perfetto has yet another implementation that is also called protozero,
    <https://perfetto.dev/docs/design-docs/protozero>
- `protos/perfetto/trace/trace.proto`
  - `Trace`
    - `packet` is a sequence of `TracePacket`
  - `TracePacket`
    - `timestamp_clock_id` and `timestamp` are the clock and timestamp of the
      packet
    - `trusted_uid` and `trusted_pid` are service-written uid/pid of the producer
    - `trusted_packet_sequence_id` is service-written id that uniquely
      identified producer+writer in this session
    - `interned_data` for data interning to reduce trace size
    - `trace_packet_defaults` provides default values to reduce trace size
    - `previous_packet_dropped`
    - `sequence_flags`
      - `SEQ_INCREMENTAL_STATE_CLEARED` means prior incremental data (e.g.,
      	`interned_data`, trace_packet_defaults`) will no longer be referenced
      	by this or future packets
      - `SEQ_NEEDS_INCREMENTAL_STATE` means this packet requires valid
      	incremental data to parse
    - `data` is one of many possible payloads, such as
      - misc
        - `ClockSnapshot`
        - `TraceConfig`
        - `CpuInfo`
        - `FtraceEventBundle`
      - track-related
	- `TrackDescriptor` describes a track
	  - that is, a row in the trace viewer
	  - there is a track for each process and thread
        - `TrackEvent` describes a track event
          - an event on a track
          - slice begin/end, instance, counter
      - gpu-related
        - `FrameTimelineEvent`
        - `GpuCounterEvent`
        - `GpuLog`
        - `GpuMemTotalEvent`
        - `GpuRenderStageEvent`
        - `GraphicsFrameEvent`
        - `VulkanApiEvent`
        - `VulkanMemoryEvent`
  - `TrackEvent`

## Host/Guest Time Sync

- Host
  - one of the producers must use modified SDK, such as perCetto, to take
    special clock snapshots
  - in `PercettoDataSource::DoIncrementalUpdate`, perCetto adds a trace packet
    with two clocks
    - one has `clock_id` `BUILTIN_CLOCK_BOOTTIME` (or
      `BUILTIN_CLOCK_MONOTONIC`) and `timestamp` from
      `clock_gettime(CLOCK_BOOTTIME or CLOCK_MONOTONIC)`
    - one has `clock_id` `kCpuCounterClockId (64)` and `timestamp` from
      `rdtsc` (x86) or `cntvct` (arm) to count cpu cycles
- Guest
  - one of the producers must use modified SDK or generate track events with
    special debug annotations
  - in `tick_forever`, `com.google.perfettoguesttimesync` generates
    `cros/guest_clock_sync` track events with debug annotations
    - `clock_sync_boottime` is from `clock_gettime(CLOCK_BOOTTIME)`
    - `clock_sync_monotonic` is from `clock_gettime(CLOCK_MONOTONIC)`
    - `clock_sync_cputime` is from `rdtsc` (x86) or `cntvct` (arm) to count
      cpu cycles
- `vperfetto_merge`
  - `getTraceCpuTimeSync` computes `TraceCpuTimeSync` using clock snapshots
  - these use the last snapshot
    - `clockId` is `BUILTIN_CLOCK_BOOTTIME` or `BUILTIN_CLOCK_MONOTONIC`
    - `clockTime` is `clock_gettime`
    - `cpuTime` is cpu cycle count
  - this use the first and the last snapshot
    - `cpuCyclesPerNano` is "cpu cycle delta" divided by "clock time delta"
