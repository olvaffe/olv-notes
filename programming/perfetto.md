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
