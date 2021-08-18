`VK_LAYER_MESA_overlay`
=======================

## How does it work?

- command buffer stats
  - on `vkCmdDraw`, it increments `draw`
  - on `vkCmdDrawIndexed`, it increments `draw_indexed`
  - on `vkCmdBindPipeline`, it increments `pipeline_graphics` or
    `pipeline_compute`
- device stats
  - on `vkQueueSubmit`, it increments `submit` 
  - it also accumulates all cmd buf stats in the device
- swapchain stats
  - on `vkAcquireNextImageKHR`, it increments `acquire` and accumulates
    `acquire_timing`
  - on `vkQueuePresentKHR`
    - it increments `frame`
    - it accumulates device stats in the swapchain
    - it accumulates `present_timing`
- gpu stats
  - many stats are CPU-based
  - many are GPU-based
  - from `vertices`, `primitives`, up to `gpu_timing` rely on GPU queries
