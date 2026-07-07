# Perfetto

## Use

- <https://perfetto.dev/docs/contributing/build-instructions>
  - `git clone https://github.com/google/perfetto.git`
  - `tools/install-build-deps [--linux-arm]` to pull dependencies
    - `BUILD_DEPS_TOOLCHAIN_HOST`
      - `gn`, `clang-format`, `ninja`, `clang`
    - `BUILD_DEPS_HOST`
      - `googletest`, `protobuf`, `abseil-cpp`
      - `libcxx`, `libcxxabi`, `libunwind`
      - `libfuzzer`, `benchmark`, `libbacktrace`
      - `sqlite`, `expat`
      - android core, unwind, logging, base, procinfo, bionic, etc.
      - `lzma`, `zstd`, `zlib`
      - `linenoise`, `bloaty`, `re2`, `pcre2`, `open_csd`
    - `BUILD_DEPS_LINUX_CROSS_SYSROOTS`: arm64 sysroot
  - `tools/gn args out/blah`
    - `target_os = "linux"`
    - `target_cpu = "x64" or "arm64"`
    - `is_debug = false`
    - `cc_wrapper = "ccache"`
  - `ninja -C out/blah`
- deploy
  - `scp out/blah/tracebox dst:~`
- Run
  - start `tracebox traced` as a regular user
  - start `tracebox traced_probes --reset-ftrace` as root
    - or, make `/sys/kernel/debug` accessible by user
  - use `tracebox perfetto` to start/stop tracing
    - `tracebox perfetto -c configs/scheduling.cfg --txt --out aaa`
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

## Architecture

- processes
  - there is a tracing service, `tracebox traced`
  - there are producers that connect to `traced`
    - `tracebox traced_probes`
    - every app that uses the tracing SDK
  - there is a consumer who can config, start, and stop tracing sessions
    - `tracebox perfetto`
- tracing data flow
  - producer
    - each producer has a shared memory with `traced`
    - a producer can have several data sources
      - `traced_probes` has ftrace, `/proc/stat`, etc. as its data sources
      - different components of an app can be difference data sources
    - each data source sub-allocates chunks from the shmem and writes
      `TracePacket` protos to chunks
      - the raw data are minimally parsed into protos
      - this is managed by a `TraceWriter`
    - producer notifies `traced` about completed chunks
      - `traced` will process the completed chunks and mark them free
      - if `traced` fails to keep up and there is no free chunk in shmem, by
        default, `TraceWriter` writes to dummy chunks which get discarded
  - traced
    - traced has a shmem with each producer for isolation
    - traced processes completed chunks on a shmem upon `CommitData` request
      - it acquires a completed chunk for reading
      - it moves `TracePacket` on the chunk to a central `TraceBuffer`
        - the number, size, fill policy, and routing are configurable
      - it releases the chunk to mark it free
    - traced streams out `TraceBuffer` protos upon `ReadBuffers` request
      - it writes the protos to the consumer over a unix socket
    - alternatively, traced can write `TraceBuffer` data to an fd if provided
      by the consumer
  - consumer
    - a consumer starts/stops a tracing session
      - it sends `EnableTracing` to traced to start
      - it sends `Flush` and `DisableTracing` to traced to stop
    - consumer sends a `ReadBuffers` request to receive `TracePacket` protos
    - consumer writes a `Trace` proto to a file
      - a `Trace` proto consists of an array of `TracpketPacket` protos
- analysis of a trace file
  - a `TraceProcessor` is a library that takes a trace file as an input,
    parses it into a in-memory SQL db, and provides SQL API to its clients
  - `TraceProcessor` supports native perfetto trace file, or other legacy
    trace files
    - android systrace
    - chrome json

## Directory Layout

- `src/tracing` and `include/perfetto/tracing` are most of the SDK
  - SDK supports in-process server and has server code
  - it has client code to connect to the system server or to the in-process
    server
  - it has `DataSource` that can be used to implement any producer
  - it has `TrackEventDataSource` that implmement a producer that can
    produce `TrackEvent`
- `src/traced/service` is `tracebox traced`
- `src/traced/probes` is `tracebox traced_probes`
- `src/profiling/perf` is `tracebox traced_perf`
- `src/perfetto_cmd` is `tracebox perfetto`
- `protos` is the protobuf definitions
  - `perfetto/trace/trace.proto` defines the protobuf for a trace file
  - `perfetto/config/trace_config.proto` defines the protobuf for a trace config file
  - `perfetto/ipc` defines the protobuf for traced/producer/consumer IPC
  - `perfetto/trace_processor` probably defines the protobuf for
    `trace_processor` and ui IPC
    - `trace_processor` loads a trace file and provide a SQL interface to
     clients
  - `perfetto/trace/perfetto_trace.proto` and
    `perfetto/config/perfetto_config.proto`
    - these are generated by concatenating all other files
    - they are for use by external projects
- `ui` is the web app

## protobuf

- implementations
  - the official one is <https://github.com/protocolbuffers/protobuf>
  - C library, <https://github.com/protobuf-c/protobuf-c>
  - minimal and high-performance, <https://github.com/mapbox/protozero>
  - perfetto has yet another implementation that is also called protozero,
    <https://perfetto.dev/docs/design-docs/protozero>
- <https://protobuf.dev/programming-guides/encoding/>
  - unsigned 64-bit ints are encoded as varints
    - a 64-bit uint takes one to ten bytes
    - each byte represents 7 bits of the uint, from LSB to MSB of the int
      - if the MSB of the byte is 0, this is the last byte
      - otherwise, there are more bytes
    - small uints take fewer bytes
  - a message is encoded as a sequence of `(tag, value)` pairs
    - a `tag` is a uint encoded as varint
      - `tag & 7` is the type of the field
      - `tag >> 3` is the id of the field
    - type 0: int32, int64, uint32, uint64, sint32, sint64, bool, enum
      - `value` is also a uint encoded as varint
    - type 1: fixed64, sfixed64, double
      - `value` has a fixed width of 64 bits
    - type 2: string, bytes, embedded messages, packed repeated fields
      - `value` is a varint, which denotes the length, followed by length
        bytes
    - type 5: fixed32, sfixed32, float
      - `value` has a fixed width of 32 bits

## `TraceConfig` proto

- `protos/perfetto/config/trace_config.proto`
  - or the generated `protos/perfetto/config/perfetto_config.proto`
- top-level `TraceConfig`
  - `buffers` configs `traced` central buffers
    - when a producuer flushes, traced copies data from the producer's shmem
      to one of the central buffers
  - `data_sources` configs data sources
  - `duration_ms` is the trace duration
    - traced will trigger `OnTracingDisabled`
  - direct write from traced
    - `write_into_file` tells `traced` to write to the provided fd
    - `output_path` is the direct write path
    - `file_write_period_ms` is the write period
    - `max_file_size_bytes` is the size limit
  - `incremental_state_config`
    - when a producer writes a trace packet, it may refer to earlier packets
      - the packet can use ids to refer to interned strings
    - with ring buffer, the earlier packets may have been overwritten
    - this tells producers to assume earlier packets have been invalidated and
      thus need to be emitted again
  - `compression_type: compression_type_deflate`
    - this happens during serialization, in response to `ReadBuffers` or
      periodically duing direct write
- `DataSourceConfig`
  - `name` identifies the data source to config
  - `target_buffer` specifies the central buffer to use
  - kernel
    - `ftrace_config` is for `linux.ftrace` from `traced_probes`
      - it collects from `/sys/kernel/tracing` for various events
      - atrace categories
        - `gfx` enables `sde`, etc.
        - `sched` enables `sched`, `cgroup`, `oom`, `task`, etc.
        - `irq` enables `irq` and `ipi`
        - `irqoff` and `preemptoff` enable `preemptirq`
        - `i2c` enables `i2c`
        - `freq` enables `power`, `clk`, `cpuhp`, etc.
        - `idle` enables `power/cpuidle`
        - `disk` enables `f2f2`, `ext4`, `block`, and `ufs`
        - `mmc` enables `mmc`
        - `fence` enables `dma_fence`
        - `workq` enables `workqueue`
        - `memreclaim` enables `vmscan`
        - `regulators` enables `regulator`
        - `binder_driver` and `binder_lock` enables `binder`
        - `pagecache` enables `filemap`
        - `memory` enables `kmem`, `gpu_mem`, etc.
        - `thermal` enables `thermal`
    - `process_stats_config` is for `linux.process_stats` from `traced_probes`
      - it polls `/proc/<pid>` for per-process stats, including names
    - `sys_stats_config` is for `linux.sys_stats` from `traced_probes`
      - it polls from `/proc` and `/sys`, including
        - `/proc/meminfo`
        - `/proc/vmstat`
        - `/proc/stat`
        - `/sys/class/devfreq`
        - `/sys/devices/system/cpuN/cpufreq`
        - `/proc/buddyinfo`
        - `/proc/diskstats`
        - `/proc/pressure`
        - `/sys/class/thermal`
        - `/sys/devices/system/cpuN/cpuidle`
        - `/proc/slabinfo`
    - `perf_event_config` is for `linux.perf` from `traced_perf`
      - it collects perf events
    - `system_info_config` is for `linux.system_info` from `traced_probes`
      - it collects these info once
        - `/proc/cpuinfo`
        - `/sys/devices/system/cpu/cpuN`
        - `/proc/interrupts`
  - distro
    - `journald_config` is for `linux.systemd_journald` from `traced_probes`
  - gpu
    - `gpu_counter_config` is for `gpu.counters` from vendor daemons
      - vendor daemons
        - android has generic `gpu_counter_producer`
          - it dlopens `libgpudataproducer.so` or
            `graphics.gpu.profiler.counter_producer_lib`
        - mesa provides both `pps-producer` and `libgpudataproducer.so`
        - mali provides both `gpudataproducer` and `libgpudataproducer.so`
      - ui visualizes them in `GPU` group, `Counters` sub-group
    - `vulkan_memory_config` is for `vulkan.memory_tracker` from vendor umds
      - mesa turnip names it `gpu.memory.msm` instead
      - ui visualizes it in `GPU` group
    - `gpu_renderstages_config` is for `gpu.renderstages` from vendor umds
      - `debug.graphics.gpu.profiler.perfetto` enables the support for some
        umds
  - chrome
    - `chrome_config` is for `org.chromium.trace_event`
    - `chromium_system_metrics` is for `org.chromium.system_metrics`
    - `chromium_histogram_samples` is for `org.chromium.histogram_samples`
  - app
    - `track_event_config` is for `track_event` from perfetto sdk
      - app embeds perfetto sdk to provide `track_event` data source
  - android
    - `android_power_config` is for `android.power` from `traced_probes`
      - it polls from `IHealth` and `IPowerStats` for power info
    - `android_log_config` is for `android.log` from `traced_probes`
      - it polls from `logd`
    - `android_polled_state_config` is for `android.polled_state` from
      `traced_probes`
      - it polls display states
    - `android_system_property_config` is for `android.system_property` from
      `traced_probes`
      - it collects from `init`
    - `statsd_tracing_config` is for `android.statsd` from `traced_probes`
      - it collects from `statsd`
    - `network_packet_trace_config` is for `android.network_packets` from
      `system_server`
    - `cpu_per_uid_config` is for `android.cpu_per_uid` from `traced_probes`
    - `user_list_config` is for `android.user_list` from `traced_probes`
    - `android_aflags_config` is for `android.aflags` from `traced_probes`
      - it invokes `/system/bin/aflags list --format proto`
  - android winscope
    - `surfaceflinger_layers_config` is for `android.surfaceflinger.layers`
      from `surfaceflinger`
    - `surfaceflinger_transactions_config` is for
      `android.surfaceflinger.transactions` from `surfaceflinger`
    - `android_input_event_config` is for `android.input.inputevent` from
      `system_server`
    - `windowmanager_config` is for `android.windowmanager` from
      `system_server`
    - `inputmethod_config` is for `android.inputmethod` from `system_server`,
      ime service, and apps

## Example Config

- `buffers`
  - `size_kb: 524288` creates a `TraceBuffer` of 512MB
  - `fill_policy: DISCARD` discards all overflow protos
- `data_sources` for `linux.system_info`
  - it collects cpu info/feats/freqs/caps
  - to query max freqs,
    `SELECT cpu, MAX(freq) FROM cpu_available_frequencies GROUP BY cpu`
- `data_sources` for `linux.sys_stats`
  - `cpufreq_period_ms` visualizes as `CPU Frequency` group
    - `power/cpu_frequency` is better, but it does not work with intel/amd pstate
    - `power/cpu_idle` is used to visualize as solid or hollow areas
  - `thermal_period_ms` visualizes as `Thermals -> Temperature (/sys)` group
    - `thermal/thermal_temperature` is better, but it typically does not work
  - `diskstat_period_ms` visualizes as `IO -> Diskstat` group
  - `meminfo_period_ms` visualizes as `Memory -> Meminfo` group
  - `vmstat_period_ms` visualizes as `Memory -> vmstat` group
  - `stat_period_ms` visualizes as
    - `System -> {IRQ,Softirq} Count` groups
    - `System -> {num_forks,num_irq_total,num_softirq_total}` tracks
    - `CPU -> {User,Nice,Kernel,Idle,IO Wait,...} Time` groups
  - `psi_period_ms` visualizes as `System -> PSI` group
  - `devfreq_period_ms` visualizes as `Hardware -> Clock Frequency` group
  - `gpufreq_period_ms` visualizes as `GPU -> GPU 0 Frequency` track
    - it only supports kgsl/i915/amdgpu and only card0
  - `cpuidle_period_ms` visualizes as `CPU -> CPU Idle {,Per Cpu} Time In State` groups
- `data_sources` for `linux.sysfs_power`
  - it visualizes as `Power` group
- `data_sources` for `linux.systemd_journald`
  - it visualizes as `Journald logs` track
- `data_sources` for `linux.perf`
  - it is provided by `tracebox traced_perf`
  - `timebase`
    - `frequency: 100` is sample frequency per "second"
    - `counter: SW_CPU_CLOCK` is the counter used to define a "second"
      - together, we sample once whenever a cpu core is busy for 10ms
    - `timestamp_clock: PERF_CLOCK_MONOTONIC` is the clock used by perf
  - `callstack_sampling`
    - `kernel_frames: true` unwinds kernel frames as well
- `data_sources` for `linux.process_stats`
  - it creates process groups to hold thread tracks and process stat tracks
  - `proc_stats_poll_ms` visualizes as `Process -> {mem.*,oom_score_adj}` tracks
    - `kmem/rss_stat` and `oom/oom_score_adj_update` are better
- `data_sources` for `linux.ftrace`
  - `ftrace_events`
    - `sched/sched_switch` visualizes as
      - `CPU Scheduling` group
      - `Scheduler` group
      - `Kernel threads` group
      - `Process -> Thread` tracks
    - `sched/sched_waking` visualizes as `Runnable` slices in thread tracks
    - `task/task_newtask` visualizes as `Runnable` slices in thread tracks,
      for new threads
    - `task/task_rename` handles renames?
    - subsets of
      - `power`, `thermal`, `devfreq`
      - `dma_fence`, `gpu_scheduler`, `drm`, `gpu_mem`
      - `irq`, `irq_vectors`, `timer`
      - `rpm`
      - `workqueue`, `vmscan`, `raw_syscalls`
      - `kmem`, `oom`
  - `syscall_events` is `raw_syscalls` filter
  - `symbolize_ksyms: true` symbolizes kernel func addrs
  - `atrace_categories` enables atrace categories
  - `atrace_apps: "*"` enables atrace app categories for all apps and system
    services
- `data_sources` for `gpu.counters`
- `data_sources` for `gpu.renderstages`
- `data_sources` for `track_event`
  - `enabled_tags: "slow"` enables slow categories
- `data_sources` for `org.chromium.trace_event`
- `data_sources` for `android.power`
- `data_sources` for `android.gpu.memory`
- `data_sources` for `android.log`
- `data_sources` for `android.surfaceflinger.frametimeline`

# `Trace` proto

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
      `interned_data`, `trace_packet_defaults`) will no longer be referenced
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
      - `GpuCounterEvent`
      - `GpuRenderStageEvent`
      - `VulkanMemoryEvent`
      - `GpuLog`
      - `VulkanApiEvent`
      - `GpuMemTotalEvent`
      - `GpuInfo`
      - `GenericGpuFrequencyEvent`
      - `GraphicsFrameEvent`
      - `FrameTimelineEvent`
- `TrackEvent`

## `tracebox traced`

- `TracingService` is the service
- `ProducerEndpoint` is a connection to a producer
- `ConsumerEndpoint` is a connection to a consumer
- for example, when `traced` wants to set up a data source,
  - `TracingServiceImpl::SetupDataSource`
  - `TracingServiceImpl::ProducerEndpointImpl::SetupDataSource`
  - `ProducerIPCService::RemoteProducer::SetupDataSource`
  - send the request to the remote producer process

## `tracebox perfetto`

- `PerfettoCmdMain` is the entrypoint
  - it is called by `tracebox perfetto` or by the legacy `perfetto` exec
  - `ParseCmdlineAndMaybeDaemonize` parses args
    - `trace_out_path_` is from `--out`
    - `trace_config_` is from `--config`
      - if `--txt`, `TraceConfigTxtToPb` converts txt to bin first
    - `OpenOutputFile` opens `trace_out_path_` and inits `trace_out_stream_`
    - `packet_writer_` points to `trace_out_stream_`
  - `ConnectToServiceRunAndMaybeNotify`
    - `ConsumerIPCClient::Connect` connects to consumer socket
    - `SetupCtrlCSignalHandler` installs ctrl-c handler
      - it calls `ConsumerEndpoint::Flush` and
        `ConsumerEndpoint::DisableTracing`
- `Consumer` callbacks
  - `OnConnect` is called when the connection to `traced` is established
    - `ConsumerEndpoint::EnableTracing` triggers tracing
  - `OnTracingDisabled` is called when tracing is disabled
    - `ConsumerEndpoint::ReadBuffers` triggers readback
  - `OnTraceData` is called when readback is triggered
    - `packet_writer_->WritePackets` writes packets to file
    - `FinalizeTraceAndExit` calls `task_runner_.Quit` to exit mainloop
  - `OnDisconnect` is called when the socket is closed
- upon ctrl-c,
  - `ConsumerEndpoint::Flush` requests producers to flush to traced
  - `ConsumerEndpoint::DisableTracing` disables tracing

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
- to convert txt config to binary config,
  - install `protobuf-compiler` to get `protoc`
  - the config proto file is `protos/perfetto/config/perfetto_config.proto`
    - <https://raw.githubusercontent.com/google/perfetto/master/protos/perfetto/config/perfetto_config.proto>
  - `protoc --encode=perfetto.protos.TraceConfig perfetto_config.proto < config.txt > config.bin`

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

## Web UI

- components
  - core, connect everything together
  - widgets, for ui elements
  - trace recorder, to initialize tracing from ui
  - trace processor engine compiled to WASM
    - a trace file consists of a `Trace` proto
    - trace processor parses the trace file into an in-memory sql db
    - alternatively, `trace_processor_shell` can listen to a local port that
      ui connects to
  - plugins, almost all analysis and visualization are provided by plugins
    - core plugins
    - domain plugins
- timeline tab
  - each row is a track
  - a group is a track that contains child tracks
  - non-standard groups
    - they all have negative priority and show at the beginning
    - `CPU Scheduling`: per-cpu schedulign
    - `CPU Frequency`: per-cpu freq
    - `Ftrace Events`: per-cpu raw ftrace events
  - standard groups
    - they all have priority 0
    - `Thermals`: thermal zones, cooling devices, etc.
    - `Power`: batteries, etc.
    - `CPU`: max/min scaling freqs, capacity, utilization, etc.
    - `Memory`: meminfo, vmstat, reclaim, etc.
    - `Hardware`: clk, devfreq, drm vblank, dma-fence, etc.
    - `GPU`: freq, mem, vulkan events
      - `Counters`: aka gpu counters
      - `Hardware Queues`: aka render stages
      - `Work Period`: android-specific
    - `Device State`: android-specific screen state, charging state, etc.
    - `IO`: block io, diskstat, ufs, f2fs, etc.
    - `Network`: tx, rx, etc.
    - `System`: irq count, psi, clock snapshot (time domain sync), etc.
    - `Kernel`: almost unused?
    - `Hypervisor` pkvm
    - `User Interaction`: android-specific end-to-end input latency
  - process groups
    - they all have priority 50 and show at the end
    - `Kernel threads`: "kernel process" for kthreads, workers, etc.
    - userspace processes
    - orphaned threads
- GPU-related custom events
  - `GpuCounterEvent`
    - producer: privileged vendor services
      - `gpu.counters` or `gpu.counters.<vendor>`
    - parser: `GraphicsEventModule` parses to counter tracks
    - ui: `dev.perfetto.Gpu` visualizes under `GPU -> Counters`
  - `GpuRenderStageEvent`
    - producer: vendor umds
      - `gpu.renderstages` or `gpu.renderstages.<vendor>`
    - parser: `GraphicsEventModule` parses to slice tracks
    - ui: `dev.perfetto.Gpu` visualizes under `GPU -> Hardware Queues`
  - `VulkanMemoryEvent`
    - producer: vendor umds
      - `gpu.memory` or `gpu.memory.<vendor>`
    - parser: `GpuEventParser` parses to counter tracks
    - ui: `dev.perfetto.Gpu` visualizes under
      - `GPU -> Vulkan Allocations `
      - `GPU -> Vulkan Binds`
      - `GPU -> Vulkan Driver Memory`
  - `GpuLog`
    - producer: ?
    - parser: `GpuEventParser` parses to a slice track
    - ui: `dev.perfetto.Gpu` visualizes as `GPU Log`
  - `VulkanApiEvent`
    - producer: vendor umds
      - `gpu.renderstages` or `gpu.renderstages.<vendor>`
    - parser: `GraphicsEventModule` parses to a slice track
    - ui: `dev.perfetto.Gpu` visualizes as `GPU -> Vulkan Events`
  - `GpuMemTotalEvent`
    - producer: `gpuservice`
      - `android.gpu.memory`
    - parser: `GpuEventParser` parses to a global counter track and
      per-process counter tracks
    - ui: `dev.perfetto.Gpu` visualizes as
      - `GPU -> GPU Memory`
      - `Process -> GPU Memory`
  - `GpuInfo`
    - producer: ?
    - parser: `SystemProbesParser`
    - ui: `dev.perfetto.Gpu` visualizes gpu names instead of `GPU <n>`
  - `GenericGpuFrequencyEvent`
    - producer: ?
    - parser: `GenericKernelParser` parses to counter tracks
    - ui: `dev.perfetto.Gpu` visualizes as `GPU -> GPU <n> Frequency`
  - `GraphicsFrameEvent`
    - producer: `surfaceflinger`
      - `android.surfaceflinger.frame`
    - parser: `GraphicsEventModule` parses to slice tracks
    - ui: `dev.perfetto.Gpu` visualizes under `GPU`
      - they show the life cycles of graphics buffers
  - `FrameTimelineEvent`
    - producer: `surfaceflinger`
      - `android.surfaceflinger.frametimeline`
    - parser: `GraphicsEventModule` parses to slice tracks
    - ui: `dev.perfetto.Gpu` visualizes as
      - `Process -> Expected Timeline`
      - `Process -> Actual Timeline`
- GPU-related ftrace events
  - `GpuFrequencyFtraceEvent`
    - producer: android-specific `power/gpu_frequency`
    - parser: `FtraceEventParser` parses to counter tracks
    - ui: same as `GenericGpuFrequencyEvent`
  - `GpuMemTotalFtraceEvent`
    - producer: `gpu_mem/gpu_mem_total`
    - parser: `FtraceEventParser` parses to a global counter track and
      per-process counter tracks
    - ui: same as `GpuMemTotalEvent`
      - if a process is idle, we don't get its `gpu_mem/gpu_mem_total`
      - we rely on `GpuMemTotalEvent` to get its gpu mem size instead
  - `GpuWorkPeriodFtraceEvent`
    - producer: android-specific `power/gpu_work_period`
      - it shows how much gpu time an uid uses for an internal
    - parser: `FtraceEventParser` parses to slice tracks
    - ui: `com.android.GpuWorkPeriod` visualizes under `GPU -> Work Period`
  - `GpuPowerStateFtraceEvent`
    - producer: android-specific `power/gpu_power_state`
      - seems powervr-only
    - parser: `FtraceEventParser` parses to a slice track
    - ui: `org.kernel.Wattson` visualizes as
      - `Wattson -> GPU Power (mW)`
      - `Wattson -> GPU Energy (J)`
  - `DrmVblank*FtraceEvent`
    - producer: `drm/drm_vblank_event*`
    - parser: `FtraceEventParser` parses to slice tracks
    - ui: `dev.perfetto.TraceProcessorTrack` visualizes under
      `Hardware -> DRM VBlank`
  - `DrmSched*FtraceEvent`
    - producer: `gpu_scheduler/drm_sched_*`
    - parser: `FtraceEventParser` parses to slice tracks
    - ui: `dev.perfetto.TraceProcessorTrack` visualizes under
      `Hardware -> DRM Sched Ring`
  - `DmaFence*FtraceEvent`
    - producer: `dma_fence/dma_fence_*`
    - parser: `FtraceEventParser` parses to slice tracks
    - ui: `dev.perfetto.TraceProcessorTrack` visualizes under
      `Hardware -> DRM Fence`
      - there are also `dma_fence_wait` under `Process`

## Perfetto SDK

- `TracingBackend` is a connection to the system server or in-process server
- `TracingMuxerImpl` is a wrapper of `TracingBackend`
- `Producer` is for when the embedder is a producer
- `Consumer` is for when the embedder is a consumer
- for example, when `traced` wants to set up a data source,
  - receive the request from the server
  - `ProducerIPCClientImpl::OnServiceRequest`
  - `TracingMuxerImpl::ProducerImpl::SetupDataSource`
  - `TracingMuxerImpl::SetupDataSource`
- when the embedder wants to generate a trace event
  - first of all, the embedder is a producer
  - the embedder calls `DataSource::Trace` with a embedder's callback to
    generate the trace event
    - the callback is called only when the data source is enabled
  - the callback is called with a `TraceContext`
    - `GetDataSourceLocked` returns a pointer to the `DataSource` object
    - `GetIncrementalState` returns a pointer to an embedder-defined struct
      associated with the data source
      - the struct is defined by subclassing `DefaultDataSourceTraits` and
       overriding `IncrementalStateType`
      - the struct stores "incremental state"
      - the struct might be freed and re-created by `GetIncrementalState`
      - to reduce trace data size, perfetto uses data interning
        - e.g., the producer can generate a string table upfront
        - future packets generated by the producer can refer to the string
          table to reduce packet size
        - this string table is known as "incremental state"
      - the incremental state can get lost in the server
        - e.g., tracing using a ring buffer and the incremental state gets
          dropped
        - when that happens, the server notifices the producer to
          re-generate the incrementa state
    - `NewTracePacket`
  - `TRACE_EVENT` and variants internally use `TrackEventDataSource`, which
    is a data source implemented by the SDK
    - it works the same way as any embedder-defined data source
    - it has the concept of "categories"
- `TracingMuxerImpl`
  - in-process backend
    - the process runs the service and consumers as well
    - used to collect trace events just for the app (e.g., chrome does this)
  - system backend
    - the process is just a producer
    - used to collect trace events in the system-wide profiling
  - `Initialize`, for each backend, calls `ConnectProducer` (e.g.,
    `SystemTracingBackend::ConnectProducer`) to create an endpoint and wraps
    it in a `TracingMuxerImpl::ProducerImpl`
  - for system backend, this connects to the system-wide producer socket.
  - `ProducerIPCClientImpl::OnServiceRequest` is called for requests from the
    service.  It calls back to `TracingMuxerImpl::ProducerImpl` for things
    such as `SetupDataSource`, `StartDataSource`, or `StopDataSource`

## Add Track Event

- `PERFETTO_DEFINE_CATEGORIES(perfetto::Category("blah"))`
  - this defines a constexpr `kCategories` array
  - and declares a `g_category_state_storage` `std::atomic<uint8_t>` array
  - and defines `kConstExprCategoryRegistry` to wrap above
  - defines a new struct type, `TrackEvent` inheriting `DataSource`
- `PERFETTO_TRACK_EVENT_STATIC_STORAGE()`
  - definitions of declarations in `PERFETTO_DEFINE_CATEGORIES`
- `perfetto::Tracing::Initialize`
  - calls `Tracing::InitializeInternal`
  - `TracingMuxerImpl::InitializeInstance` is called with `InitializedMutex`
    locked
    - `instance_` is statically-initialized `TracingMuxerFake::Get()`
    - `TracingMuxerImpl::TracingMuxerImpl` sets `instance_` to itself
    - it spawns `task_runner_` to call `TracingMuxerImpl::Initialize` and
      `TracingMuxerImpl::AddBackends`
    - `TracingMuxerImpl::AddProducerBackend` calls
      `SystemProducerTracingBackend::ConnectProducer`
    - `GetProducerSocket` returns `/run/perfetto/traced-producer.sock`
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
