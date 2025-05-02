Dota 2
======

## Benchmark

    STEAM="$HOME/.steam/steam"
    RT="$STEAM/ubuntu12_32/steam-runtime/run.sh"
    DOTA="$STEAM/steamapps/common/dota 2 beta"
    
    "$RT" "$DOTA"/game/dota.sh \
	+engine_experimental_drop_frame_ticks 1 \
	+@panorama_min_comp_layer_dimension 0 \
	-prewarm_panorama \
	-vulkan \
	-fullscreen \
	-nosound \
	-autoconfig_level 3 \
	-h 900 \
	-w 1440 \
	-high \
	+timedemo_start 50000 \
	+timedemo_end 50500 \
	+timedemo dota2-pts-1971360796.dem \
	+demo_quitafterplayback 1 \
	+fps_max 0

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
