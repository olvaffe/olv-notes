Mesa utrace
===========

## Overview

- `struct u_trace`
  - the goal is to have tracepoints for gpu cmdstreams
  - when a driver builds a cmdstream, it calls `u_trace_append` (indirectly)
    to add the tracepoint payloads to `u_trace`.
  - `u_trace_append` also inserts gpu timestamp record cmd to the gpu
    cmdstream
  - later, when the cmdstream is submitted to GPU, `u_trace_flush` is called
    to flush the pyaloads to `u_trace_context`
- `struct u_trace_context`
  - there should be one `struct u_trace_context` per logical gpu ring
  - `u_trace_context_process` processes the payloads flushed
    - there is a `traceq` thread processing the payloads
    - for each payload, `traceq` waits until the gpu timestamp is available
    - if perfetto is enabled, `traceq` sends the gpu timestamp and the payload
      to perfetto
- there are several ways to enable `u_trace`
  - `GPU_TRACE=1` or `GPU_TRACEFILE=<file>`
    - this enables `u_trace`
    - it outputs to stdout or the specified file
  - `GPU_TRACE_INSTRUMENT=1`
    - this also enables `u_trace`
    - but there is no output
    - this is useful with vulkan plus perfetto.  It makes sure a vk cmdbuf
      recorded before perfetto tracing includes `u_trace` payloads
  - perfetto tracing
    - perfetto can enable/disable `u_trace` on-demand
    - when the env vars above are specified, `u_trace` is never disabled

## Example: turnip

- using turnip's `start_render_pass` as an example
- `u_trace_context_init` is called per `VkDevice`
- `u_trace_init` is called per `VkCommandBuffer`
- `start_render_pass` is a tracepoint defined by `tu_tracepoints.py`
  - the script generates a `trace_start_render_pass` function
  - `trace_start_render_pass` calls `u_trace_append` to generate the payload
    and insert a GPU timestamp read cmd to `VkCommandBuffer`
  - `traceq` waits for GPU execution.  When the timestamp is finally
    available, it calls `tu_start_render_pass` which calls `stage_start`

## Example: anv

- `anv_device_utrace_init` calls `u_trace_context_init` per-device
- `anv_create_cmd_buffer` (and `anv_cmd_buffer_reset`) calls `u_trace_init`
  per-cmdbuf
- `genX(CmdDraw)` calls `trace_intel_begin_draw` per-draw
  - if the tracepoint is enabled, `u_trace_append` is called to add the
    payload to `u_trace`
- `anv_device_utrace_flush_cmd_buffers` calls `u_trace_flush`
  - this happens at queue submit time
  - it moves tracepoint payloads from `u_trace` to `u_trace_context`
  - if a cmdbuf does not have `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`,
    its tracepoint payloads need to be copied
- `anv_device_utrace_finish` and `anv_QueuePresentKHR` call
  `u_trace_context_process`
