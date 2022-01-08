SurfaceFlinger
==============

## Frame Trace View 

- this is Unity on ARCVM
- frame pacing depends on vsyncs
  - `IRQ (arc_timer)` fires every 16.6ms
  - SF `TimerDispatch` thread wakes up, which wakes up `app` thread and
    `sf`thread
    - it also flips `VSYNC-app` and `VSYNC-sf` events
  - SF `app` thread wakes up APP `UI thread` to `Choreographer#doFrame`
    - for Unity, it also wakes up APP `UnityChoreograp` to
      `Choreographer#doFrame`
- rendering depends on fences
  - APP `UnityGfxDeviceW` thread calls `dequeueBuffer` and waits until the
    buffer is idle
  - It then renders to the buffer and does a `eglSwapBuffers` to `queueBuffer`
  - If the buffer is idle by next vsync, SF latches it
  - when systrace is enabled, APP `HWC release and `GPU completion` threads
    are created
    - APP `HWC release` tracks all out-fences returned in `dequeueBuffer`
    - APP `GPU completion` tracks all in-fences upon `queueBuffer`
- at perfectly stable 60fps,
  - the three buffers of a buffer queue are
    - one is owned by HWC
    - one is owned by SF to be latched
    - one is in the buffer queue
  - on vsync,
    - SF latches and sends the buffer it owns to HWC
    - the one owned by HWC is returned to the buffer queue
    - app dequeues from the queue to render the next frame and passes it to SF
- a GPU-bound app
- a CPU-bound app
