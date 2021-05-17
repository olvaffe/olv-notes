Dota 2
======

## Life of a Frame from System's View

- these are just my guess
- `VKRenderThread` calls `vkQueuePresentKHR` to present the current frame and
  moves on to the next frame immediately
- `VKRenderThread`, before making too much progress, notices that the frame
  data associated with the next frame are still being used by GPU and calls
  `vkWaitForFences` before they can be recycled
  - after `vkQueuePresentKHR`, `VKRenderThread` works for like 0.1ms
  - the wait takes 50ms at 20fps
- while `VKRenderThread` waits, the other threads update the game states and
  start preparing the resources and command buffers for the next frame
  - the main thread records some command buffers and allocates/updates some
    descriptor sets
  - the GlobPool threads do the same
  - pipelines might need to be created
- after `VKRenderThread` resumes, it checks some `vkGetQueryPoolResults`,
  creates some command buffers, and waits the main and GlobPool threads
  - normally, main and GlobPool threads are done by then and this takes <1ms
  - but if main or GlobPool threads are still busy, such as busy in
    `vkCreateGraphicsPipelines` which takes up to hundreds of ms,
    `VKRenderThread` waits
- `VKRenderThread` creates some more command buffers, wakes up the main thread
  to `vkAcquireNextImageKHR` and continue to create even more command buffers
  - the swapchain is in immediate mode and the main thread spends like 0.1ms
  - `VKRenderThread` spends like 1.5ms to 3.5ms in total and makes a single
    `vkQueueSubmit`
- `VKRenderThread` calls `vkQueuePresentKHR` again
