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

## Example

- using turnip's `start_render_pass` as an example
- `u_trace_context_init` is called per `VkDevice`
- `u_trace_init` is called per `VkCommandBuffer`
- `start_render_pass` is a tracepoint defined by `tu_tracepoints.py`
  - the script generates a `trace_start_render_pass` function
  - `trace_start_render_pass` calls `u_trace_append` to generate the payload
    and insert a GPU timestamp read cmd to `VkCommandBuffer`
  - `traceq` waits for GPU execution.  When the timestamp is finally
    available, it calls `tu_start_render_pass` which calls `stage_start`
