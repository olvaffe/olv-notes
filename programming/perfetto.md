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
