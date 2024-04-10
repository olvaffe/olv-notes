Mesa Perfetto
=============

## Protos

- `GpuCounterConfig`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/config/gpu/gpu_counter_config.proto>
  - `counter_period_ns` is the sample period
  - `counter_ids` is counters to sample
  - `instrumented_sampling` is to sample on events (queue submit,
    end-of-renderpass) rather than using the sample period
  - `fix_gpu_clock` is to lock the gpu freq
- `GpuCounterDescriptor`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/common/gpu_counter_descriptor.proto>
  - `specs`, an array of `GpuCounterSpec`
    - id, name, desc, peak value, unit, default, groups
  - `blocks`, an array of `GpuCounterBlock`
    - a block is an abstraction of a function block of the gpu hw
    - id, name, desc, capacity, counter ids
  - `min_sampling_period_ns` and `max_sampling_period_ns`
  - `supports_instrumented_sampling`
- `GpuCounterEvent`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/trace/gpu/gpu_counter_event.proto>
  - `counters`, an array of `GpuCounter`
    - `counter_id` and `value`
  - `gpu_id` for multi-gpu
- `GpuRenderStageEvent`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/trace/gpu/gpu_render_stage_event.proto>
  - `event_id`, `hw_queue_iid`, `stage_iid`, `gpu_id`
  - `duration`
  - `context`, `render_target_handle`, `submission_id`
  - more
- `GpuLog`
  - `severity`, `tag`, `log_message`
- `VulkanApiEvent`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/trace/gpu/vulkan_api_event.proto>
  - `event`, usually a vk queue submit call
- `VulkanMemoryEvent`
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/master/protos/perfetto/trace/gpu/vulkan_memory_event.proto>
  - for vk memory alloc/free/bind calls

## `pps-producer`

- PPS stands for Perfetto ProducerS, I guess
- `pps-producer`
  - enabled with `-Dperfetto=true`
  - requires root to run
- `pps-config`
  - depends on `libdocopt`
  - can print counter groups, names, values, etc.
- initialization
  - `Driver::default_driver_name`
    - `Driver::supported_device_names` returns all supported drivers
      - this calls ctor of all `Driver`
    - `DrmDevice::create_all` opens all drm rendernodes and wraps them in
      `DrmDevice`
    - returns the name of the first device that has a driver for
  - `GpuDataSource::register_data_source`
    - set the name to `gpu.counters.<drv>`
    - `perfetto::DataSource::Register`
- mainloop
  - `GpuDataSource::wait_started`
    - it blocks until the perfetto thread receives a start event
  - `DataSource::Trace` calls `GpuDataSource::trace_callback` if the data
    source is enabled
- perfetto thread
  - `GpuDataSource::OnSetup`
    - `Driver::init_perfcnt`
    - `Driver::enable_counter` or `Driver::enable_all_counters` depending on
      if `counter_ids` is specified in the config
    - set `time_to_sleep` to max of
      - `MIN_SAMPLING_PERIOD_NS` (50us)
      - `Driver::get_min_sampling_period_ns`
      - `counter_period_ns` config
  - `GpuDataSource::OnStart`
    - `Driver::enable_perfcnt`
    - wakes up the main thread
  - `GpuDataSource::OnStop`
    - `close_callback` adds the last packet that is empty, and
      `TraceContext::Flush`
    - `Driver::disable_perfcnt`
- `GpuDataSource::trace_callback` and `GpuDataSource::trace`
  - `time_to_sleep` is the sample period
  - `time_to_trace` is how long the last `GpuDataSource::trace` call took
  - sleeps if necessary and then `GpuDataSource::trace`
    - if incremental state was cleared, it needs to be resend
      - a packet with `SEQ_INCREMENTAL_STATE_CLEARED`
      - a packet with `GpuCounterEvent`
        - just `GpuCounterDescriptor` based on `Driver::groups` and
          `Driver::enabled_counters` to describe counters
        - no counter values
      - another packet for cpu/time timesync
        - `Driver::gpu_clock_id`
        - `Driver::gpu_timestamp`
    - `sched_setscheduler` to RT
    - `Driver::dump_perfcnt`, `Driver::next`, and `Counter::get_value`
    - every `CORRELATION_TIMESTAMP_PERIOD` (1s), add a cpu/gpu timesync packet
    - `sched_setscheduler` to non-RT

## PPS core

- `DrmDevice::device_count`
- `Driver`
- `CounterGroup`
  - has `name` and `id`
  - has an array of `counters` and an array of `subgroups`
    - thus them form a tree
- `Counter`
  - `id`, `name`, and `group`
  - `offset` and `derived`
- `DrmDevice`
  - `device_count`, `create`, and `create_all`
    - static methods for enumeration
  - `fd`, `gpu_num`, and `name`
- `Driver`

## Intel

- `IntelDriver::init_perfcnt`
  - initialize `groups` and `counters` and more
- `IntelDriver::enable_counter`
  - add counter to `enabled_counters`
- `IntelDriver::get_min_sampling_period_ns`
  - return how fast HW sampling can be
- `IntelDriver::enable_perfcnt`
  - `DRM_IOCTL_I915_PERF_OPEN` to start HW sampling
- `IntelDriver::dump_perfcnt`
  - read all raw HW sampling data
- `IntelDriver::next`
  - parse the next raw HW sampling data into `intel_perf_query_result`
- `Counter::getter`
  - an anonymous function in `IntelDriver::init_perfcnt`
  - read values from `intel_perf_query_result`
- `intel_perf_init_metrics(..., false, false)`
  - `oa_metrics_available`
    - `i915_query_supported` is optional and is unused
      - `DRM_IOCTL_I915_QUERY(DRM_I915_QUERY_PERF_CONFIG, DRM_I915_QUERY_PERF_CONFIG_LIST)`
    - `I915_PARAM_PERF_REVISION`
      - it is 5 right now
    - must be root or `/proc/sys/dev/i915/perf_stream_paranoid` must be 0
      - `sysctl dev.i915.perf_stream_paranoid=0`
    - `perf->sysfs_dev_dir` is initialized to `/sys/dev/char/226:128/device/drm/card0`

## freedreno

- `FreedrenoDriver::Countable`
  - every call to `FreedrenoDriver::countable` adds a `Countable`
  - countables are tracked in `countables`
  - their ids are from `next_countable_id`
  - these are hidden from pps core
  - internally, each `FreedrenoDriver::Countable` is associated with a
    `fd_perfcntr_countable` and a `fd_perfcntr_counter`
    - a `fd_perfcntr_countable` is an event that can be counted
    - a `fd_perfcntr_counter` is a hw sampler that counts an event at a fixed
      rate
      - e.g., we can program a `fd_perfcntr_counter` to count
      	`PERF_CP_ALWAYS_COUNT` event
      - this gives us gpu ticks
    - a `Countable` uses a `fd_perfcntr_counter` to count a
      `fd_perfcntr_countable`
  - `resolve`
    - it associates a `Countable` with a `fd_perfcntr_countable`
      and `fd_perfcntr_counter` based on the event name (e.g.,
      `PERF_CP_ALWAYS_COUNT`)
  - `configure`
    - emits pm4 packets to initialize the associated `fd_perfcntr_counter`
  - `collect`
    - reads and saves `fd_perfcntr_counter` value
    - it save the last two values
  - `get_value`
    - return the delta of the last two counter values
- `FreedrenoDriver::DerivedCounter`
  - every call to `FreedrenoDriver::counter` adds a `DerivedCounter`
    - it is a subclass of `pps::Counter`
  - counters are tracked in `counters`
  - their ids are from `next_counter_id`
  - these are what pps core sees
  - when pps core calls `pps::Counter::get_value`, it calls an anonymous
    function associated with the `DerivedCounter`
    - the anonymous funcion calls `FreedrenoDriver::Countable::get_value` from
      one or more of the countables, does some ALUs, and returns a value
      sensible for UI
- `FreedrenoDriver::init_perfcnt`
  - initializes `countables` and `counters`
  - mmaps `/dev/mem`
  - calls `FreedrenoDriver::Countable::configure` on all `countables`
  - calls `FreedrenoDriver::Countable::collect` on all `countables`
    - this is to make sure `Countable::get_value` has a delta to return after
      every `Countable::collect` call
- `FreedrenoDriver::enable_counter`
  - because we don't know which `DerivedCounter` depends on which
    `Countable`s, we cannot make use of this but have to enable all
    `Countable`s
- `FreedrenoDriver::get_min_sampling_period_ns`
  - randomly returns 100us
- `FreedrenoDriver::enable_perfcnt` is nop
- `FreedrenoDriver::dump_perfcnt`
  - if the gpu was suspended, we reinitialize and return false
  - otherwise, `collect_countables` calls `collect` on all `countables`
  - `last_dump_ts` is the time of the last `collect_countables` call
  - `time` is the duration between this and last `collect_countables` calls
- `FreedrenoDriver::next`
  - returns `last_capture_ts`
