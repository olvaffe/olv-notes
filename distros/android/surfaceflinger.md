Android SurfaceFlinger
======================

## Protocol Overview

- `ISurfaceComposer` is the interface of `SurfaceFlinger`
  - `ISurfaceComposerClient` is the interface of a connection to
    `SurfaceFlinger`.  It can be created from `ISurfaceComposer`
    - `ISurface` is the interface of a on-screen surface.  It can be created
      from `ISurfaceComposerClient`.
      - `ISurfaceTexture` is the interface of the buffer storage of the
        surface.  It can be obtained from `ISurface`.
  - `IGraphicBufferAlloc` is the interface of a buffer allocator.  It can be
    created from `ISurfaceComposer`.  Buffers are allocated from this
    interface.
- This is how things work
  - an app creates `ISurfaceComposer` to talk to `SurfaceFlinger`.  It
    creates a connection, that is, `ISurfaceComposerClient`.  `ISurface`s can
    be created from the connection.  `ISurfaceTexture`s can be used to access
    surface buffers.
  - the app can also use `IGraphicBufferAlloc` to allocate offscreen buffers.

## Object Overview

- `SurfaceComposerClient` wraps `ISurfaceComposer` and
  `ISurfaceComposerClient`.
- `SurfaceControl` wraps `ISurface`.
- `Surface` wraps `SurfaceTextureClient` which wraps `ISurfaceTexture`
  - `Surface` is an old object, which becomes very thin after the introduction
    of `SurfaceTextureClient`
- `SurfaceTexture` wraps `ISurfaceCompoer` and `IGraphicBufferAlloc`.
  - It is used by clients to allocate offscreen buffers.  EGL can render to
    them, by wrapping `SurfaceTexture` in a `SurfaceTextureClient`.  Or sample
    from them, by using `EGLImage`.
  - It is also used by the server to allocate buffers and implement
    `ISurfaceTexture`
  - Because it implements `ISurfaceTexture`, it can be wrapped in
    `SurfaceTextureClient`

## EGL native types

- `FramebufferNativeWindow` is an `ANativeWindow`.  It is server-only.
- `SurfaceTextureClient` is an `ANativeWindow`.  It is client-only.
  - Thus, `Surface` and `SurfaceTexture` can both be viewed as native types
- `GraphicBuffer` is an `ANativeWindowBuffer`.  It is used by both clients and
  the server.

## Jave-level objects

- Java-level `ViewRootImpl` has a Java-level `Surface`, which maps to
  C++-level `Surface`.
- Java-level `SurfaceView` also has a Java-level `Surface`.
- Java-level `TextureView` has a Java-level `SurfaceTexture`, which maps to
  C++-level `SurfaceTexture.

## `DisplaySurface`

- has two implementations: `FramebufferSurface` and `VirtualDisplaySurface`
  - a `FramebufferSurface` is created for each internal or externail physical
    display
  - a `VirtualDisplaySurface` is created for each virtual display
- `beginFrame` is called when SF is about to render a new frame
- `prepareFrame` is called after SF knows whether client composition is
  necessary 
- `compositionComplete` is called after all layers are rendered (but before SF calls `eglSwapBuffers`)
- `advanceFrame` is called after all layers are rendered (and after SF calls `eglSwapBuffers`)
- `onFrameCommitted` is called after the new frame is presented/posted
- `FramebufferSurface` is implemented on top of a `IGraphicBufferConsumer`
  - `beginFrame`, `prepareFrame`, and `compositionComplete` are no-ops
  - `advanceFrame` acquires the next buffer in BQ and makes it the client
    target in HWC.  If the buffer is different from the previous one, the
    previous one is marked as pending release. 
  - `onFrameCommitted` releases the previous buffer.  It gets the present
    fence from HWC and associates the fence with the previous buffer.  This
    way, the previous buffer is not reused until the fence is signaled.
  - this requires `SurfaceFlinger::maxFrameBufferAcquiredBuffers` to be at
    least 2

## `DisplayDevice`

- a `DisplayDevice` is created for each display
  - it is created with an HWC display, a `DisplaySurface`, an IGBP, and an
    optinal `EGLConfig` (no config when `EGL_KHR_no_config_context` is
    supported)
  - it will create a `Surface` from the IGBP, and an `EGLSurface` from the
    surface.

## `EGLImage` for client target

- `DisplayDevice` owns N buffers and create an `EGLImage` for each of the
  buffer.
- It communicates with `DisplaySurface` which image is current

## BQ Fencing

- If supported, a producer can queue a buffer with a fence
  - gralloc unlock may return a fence
  - `eglSwapBuffers` may queue the buffer with a fence
  - `vkQueuePresentKHR` may queue the buffer with a fence
- the consumer will acquire the buffer with the associated fence
  - it will give the fence to HWC along with the buffer
  - it will also bind the buffer to a GL texture.  It needs to perform a
    GPU `eglWaitSyncKHR` or CPU `sync_wait` on the fence before the GL texture
    can be used
- the consumer can release the buffer with a different fence as well
  - HWC may return a fence for the buffer
  - GLES composition may have a fence for the client target
  - Additionally, `GLConsumer` needs to, dpending on driver support,
    - create another fence for the outstanding GL commands, or
    - create EGLSync for the outstanding GL commands
- when the producer dequeue the released buffer again
  - the consumer release fence will be returned with the buffer
  - if there is an EGLSync outstanding, dequeue needs to block until
    `eglClientWaitSyncKHR` returns

## dumpsys

- `adb shell dumpsys SurfaceFlinger`
- `adb shell dumpsys SurfaceFlinger --timestats -enable -clear`
  - this enables and clears timestats, for fps and etc.
  - one can periodically `-dump -clear` to dump and clear the stats

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
