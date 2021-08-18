Hollow Knight
=============

## System Analysis

- Threads
  - 1 main thread
  - 6 unnamed threads
    - one keeps calling `poll` (blocks for 10~13ms), `sendmsg`, `fcntl`,
      `getpid`, etc.
    - one keeps calling `futex` and `ioctl`
    - two keep calling `clock_nanosleep` to sleep 10ms
    - one keeps calling `futex` (blocks for 10~13ms) and `getpid`
    - one keeps calling `poll` which easily blocks for seconds
  - 1 `threaded-ml`
    - keeps calling `poll` (blocks for 10~13ms), `readv`, and `futex`
  - 7 `Job.Worker`
    - blocked in `futex` waiting for jobs
  - 1 `UnityGfxDeviceWorker`
- At menu, frame is 6.5ms
  - `UnityGfxDeviceWorker` blocks in `vkQueuePresentKHR` for 3ms
    - GPU bound
    - `VK_PRESENT_MODE_IMMEDIATE_KHR` with 3 images
    - when presenting, Mesa WSI does a empty submit to mark the image bo busy
      and set up a fence before sending the image bo to X11
    - X11 schedules a blit and returns the image back
    - at the 4th frame, the first image bo is reused
      - Mesa inserts `vkWaitForFences` to wait for the 1st frame
      - without this or other form of throttling, the app can get greatly
      	ahead of X11
      - indeed, when the wait is removed, the game queues frames much faster
      	than the GPU can renderer, creating a huge latency visually.  Once in
      	a while, `vkQueuePresent` takes ~100ms (blocked in the kernel driver?)
      	waiting for the rendering to catch up.
  - when it wakes up, it wakes other threads
    - one of the unnamed thread
    - the main thread
      - the main thread queues jobs for `Job.Worker`s
    - Xorg
  - it does 3 `vkGetFenceStatus`, 1 `vkResetCommandPool`, and blocks in futex
    waiting for the main thread (to update the frame data?)
  - when it wake up again, it does 10-ish `vkBeginCommandBuffer` and
    `vkEndCommandBuffer`, followed by a `vkAcquireNextImageKHR`, followed by 6
    more cmdbuf begin/end.
  - it takes 2ms by this point
  - the thread then does a `vkResetFences` and spends 1.5ms build a final cmdbuf
  - it finally calls 1 `vkQueueSubmit`, 1 `vkGetFenceStatus`, 1
    `vkResetCommandPool`, and 1 `vkQueuePresentKHR`
- In game, frame is 13ms
  - similar to in menu, excepts that `UnityGfxDeviceWorker` blocks in
    `vkQueuePresentKHR` for 10ms
    - GPU bound
