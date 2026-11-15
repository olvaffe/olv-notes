# Mesa PanVK DataSource

## `libpanfrost_perf`

- `pan_perf_create`
  - `pan_kmod_perf_create` calls `panthor_kmod_perf_create`
  - `mali_perf_get_info_for_gpu` returns the generated `mali_perf_gpu_info`
    - which defines derived counters on top of raw counters
- `pan_perf_enable` calls `panthor_kmod_perf_enable`
  - `perf_cmd_setup` creates the kernel session
  - `perf_cmd_start` starts the kernel session
- `pan_perf_dump` calls `panthor_kmod_perf_dump`
  - `panthor_perf_sample` requests and waits for a manual sample
- `pan_perf_counter_read_sum` reads a derived counter val
  - it calls `panthor_kmod_perf_get_hw_counter_value` to read a raw counter
- `pan_perf_disable` calls `panthor_kmod_perf_disable`
  - `panthor_perf_stop` stops the kernel session and waits for the final sample
- `pan_perf_destroy` calls `panthor_kmod_perf_destroy`
  - `panthor_perf_stop` stops on demand
  - `perf_cmd_teardown` destroys the kernel session
  - it destroys the session
