Fossilize
=========

## Captures and Replays

- capture with apis
  - `Fossilize::create_database` to create/open a `.foz` database
    - it returns a `Fossilize::DatabaseInterface` to manipulate the database
  - `StateRecorder recorder; recorder.init_recording_thread(db)` to initialize
    the recorder
  - from this point on, call `record_*` such as `record_application_info` or
    `record_sampler` to record objects to the db
  - when done, destroy the record to flush and stop the recording thread
- replay with apis
  - define app-specific `ReplayerInterface`
  - `StateReplayer replayer; replayer.parse(iface, db, ...)` to replay the
    database
- capture with `VK_LAYER_fossilize`
  - automatic create a db and invoke `StateRecorder`
  - `FOSSILIZE_DUMP_PATH_ENV`

## Database

- `fossilize-convert-db` unpacks a `.foz` database to (many) json files
- it looks like each `StateRecorder::record_*` call records a `Vk*Info` to a
  json snippet, and is saved to the database

## `fossilize-replay`

- a `fossilize-replay` process can be in one of the four modes
  - when `--progress` is specified, it runs in progress mode with
    `run_progress_process` being the main loop.  In this mode, it uses
    `ExternalReplayer` to start another `fossilize-replay` process with
    `--master-process`
  - when `--master-process` is specified, it runs in multi-process mode with
    `run_master_process` being the main loop.  `--num-threads` determines the
    number of child processes to fork.  Each child process uses
    `run_slave_process` as the main loop.
  - when `--slave-process` is specified, it runs in slave mode with
    `run_slave_process` being the main loop.  This is not used on Linux.  In
    multi-process mode, `fossilize-replay` forks and calls `run_slave_process`
    directly.
  - otherwise, it runs in normal mode with `run_normal_process` being the main
    loop.  `--num-threads` determines the number of threads of
    `ThreadedReplayer`
- when in progress mode, `ExternalReplayer` is used to start another
  `fossilize-replay` process in multi-process mode
  - `ExternalReplayer::Impl::start` sets up the shmem fd and the control fd
    that are used to control the external replayer and receive progress data
  - it specifies `--master-process`, `--quiet-slave`, `--shmem-fd`, and
    `--control-fd`
  - it forwards `--spirv-val`, `--num-threads`, `--on-disk-*`, and many other
    arguments
- when in multi-process mode,
  - `--control-fd` to specify a control fd
    - it can be used to control the master process
    - commands are handled by `handle_control_command`
    - `RUNNING_TARGET <N>` cotrols the number of child processes running
  - `--shmem-fd` to specify a shmem fd
    - it is used to hold a `SharedControlBlock` and a ring buffer
    - used for progress reporting from child processes
- when in slave mode,
  - it sets up handshakes with the master process and disables some signals
  - it then calls `run_normal_process`
- when in normal mode,
  - a `StateReplayer` is created to parse the database
  - as `StateReplayer` parses, recorded Vulkan states are replayed
    - e.g., `Fossilize::StateReplayer::Impl::parse_application_info` calls
      `ThreadedReplayer::set_application_info`
  - the main thread replays these states first
    - `RESOURCE_APPLICATION_INFO`
    - `RESOURCE_SAMPLER`
    - `RESOURCE_DESCRIPTOR_SET_LAYOUT`
    - `RESOURCE_PIPELINE_LAYOUT`
    - `RESOURCE_RENDER_PASS`
  - `start_worker_threads` is called
  - these states are collected as well
    - `RESOURCE_GRAPHICS_PIPELINE`
    - `RESOURCE_COMPUTE_PIPELINE`
  - `enqueue_deferred_pipelines` is called to convert the state replays into
    `EnqueuedWork`.
  - `enqueue_work_item` is called indirectly by `EnqueuedWork`s and the worker
    threads start replaying
  - `sync_worker_threads` waits for all worker threads to complete
- pipeline cache
  - by default, `fossilize-replay` warms up the driver's internal disk cache
    - it creates some `VkPipelineCache`s for pipeline replays
    - after replays, it destroys the caches
    - it does not save the pipeline caches anywhere
  - `--on-disk-pipeline-cache` saves the pipeline cache to a file
    - when specified, it creates a `VkPipelineCache` from the file
    - when done, it calls `vkGetPipelineCacheData` and saves the data back to
      the file
- crash report
  - in multi-process mode, when a child process crashes, `crash_handler` is
    called.
  - the handler uses a pipe to tell the master process it has crashed, and
    which pipeline was being replayed
  - `--timeout-seconds` specifies a timeout when stuck
    - by default, `sync_worker_threads` waits until all worker threads are
      done
    - when specified, and if no pipeline is replayed during the timeout value,
      `timeout_handler` is called.  This sends `SIGABRT` to the worker thread
      and triggers `crash_handler`
- misc options
  - `--replayer-cache` specifies a replay cache
    - it is a db that contains the hashes of replayed pipelines
    - this allows the replayed pipelines to be skipped on following runs
