# Android SurfaceFlinger

## Native Activity

- before looking into how surfaceflinger works, let's look into how a native
  activity interacts with sf
  - upon `onNativeWindowCreated`, app receives an `ANativeWindow`
  - `ANativeWindow_lock` locks the win for cpu access
  - `ANativeWindow_unlockAndPost` unlocks and presents the updated contents
  - egl and vk also accept `ANativeWindow` for gpu access
- internally, when an activity is created,
  - a `SurfaceControl` is created to connect to the sf
  - it also creates a surface for drawing, at which point
    `onNativeWindowCreated` is called
- inside libgui,
  - a `SurfaceControl` owns
    - a `SurfaceComposerClient` which is the connection to sf
    - a `SurfaceControl` which is a child surface created from sf
    - a `BLASTBufferQueue` that wraps the child surface
    - a `Surface` that wraps the blast bq and is a `ANativeWindow`
  - `ANativeWindow_lock` calls `Surface::lock` to dequeue a buffer from
    `BBQBufferQueueProducer` and locks it for cpu access
    - the buffer is either reused or newly allocated directly
  - `ANativeWindow_unlockAndPost` calls `Surface::unlockAndPost` to unlocks
    the buffer and queues it to `BBQBufferQueueProducer`
    - `SurfaceComposerClient::Transaction::setBuffer` adds buffer change to
      the transaction
- on the sf side,
  - `SurfaceComposerAIDL::createConnection` creates a `Client`
  - `Client::createSurface` creates a `Layer`
  - `SurfaceFlinger::setTransactionState` handles a batch of transactions
    - client batches their transactions
    - this function queues up transactions and notifies the scheduler
  - `SurfaceFlinger::commit` commits transactions
    - this is called from the scheduler
    - `SurfaceFlinger::updateLayerSnapshots` applies the transactions
      - `Layer::setBuffer` handles buffer change
  - `SurfaceFlinger::composite` composites a frame
    - this is also called from the scheduler
    - `CompositionEngine::present` composites and presents
      - this calls `Output::present` of each output
        - `Output::presentFrameAndReleaseLayers`
        - `Display::presentFrame`
          - note that a `Display` is a `Output`
        - `HWComposer::presentAndGetReleaseFences`
        - `HWC2::impl::Display::present`

## GPU Composition

- `adb shell service call SurfaceFlinger 1008 i32 1` forces gpu composition
- during init, `SurfaceFlinger::processDisplayAdded` adds a `DisplayDevice`
  - it creates a `LegacyFramebufferSurface` which is like a bq
  - it creates a `DisplayDevice` over the legacy fb surface
    - `Display::createRenderSurface` to create a `RenderSurface` where
      - `mNativeWindow` is the producer end of the fb
      - `mDisplaySurface` is the consumer end of the fb
- `Output::present`
  - `Output::dequeueRenderBuffer` dequeues a buf from fb
  - `Output::composeSurfaces` performs gpu composition to the buf
    - `RenderEngine::drawLayers` draws
  - `RenderSurface::queueBuffer` queues the buf
  - `LegacyFramebufferSurface::advanceFrame` calls
    `HWComposer::setClientTarget`

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
  C++-level `SurfaceTexture`.

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
  - when systrace is enabled, APP `HWC release` and `GPU completion` threads
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

## Vulkan

- Android vulkan loader implements `VK_KHR_android_surface` and `VK_KHR_swapchain`
- `vkCreateAndroidSurfaceKHR`
  - it creates a `VkSurfaceKHR` from a `ANativeWindow`, which is from
    `ANativeWindow_fromSurface`
  - internally, the concrete type is `Surface` which is a wrapper over
    `IGraphicBufferProducer`
- `vkQueuePresentKHR` calls `ANativeWindow::queueBuffer`
  - it calls `Surface::queueBuffer` which calls
    `BpGraphicBufferProducer::queueBuffer`
    - `BpGraphicBufferProducer` is a proxy object in the app process
    - `BufferQueueProducer` is the real object in the sf process
      - `BufferQueueProducer` implements `BnBufferQueueProducer`
      - `BnBufferQueueProducer::onTransact` calls
        `BufferQueueProducer::queueBuffer`
  - `BufferQueueProducer::queueBuffer` adds the buffer to the queue
    - it has a `waitForever` on the previous buffer to throttle
  - when tracing is on, after `BufferQueueProducer::queueBuffer` returns,
    `Surface::queueBuffer` adds the fence to `gpuCompletionThread` in the app
    process
    - there is a `GPU completion` thread that waits on the fence
    - the thread has a `GPU completion` counter that tracks the number of
      fences pending in the thread
- `vkAcquireNextImageKHR` calls `ANativeWindow::dequeueBuffer`
  - similarly, it calls `BufferQueueProducer::dequeueBuffer`

## Layer

- A layer has `mVertices`.  It is layer's geometry converted to 16.16 fixed point.
  Its values are

    mVertices[0] = (x, y)
    mVertices[1] = (x, y+h)
    mVertices[2] = (x+w, y+h)
    mVertices[3] = (x+w, y)
  where `w` and `h` are positive.  It uses X-window coordinates.
- In `LayerBase::drawWithOpenGL`, `mVertices` is used directly as vertex array.
  Since OpenGL uses cartesian coordinates, the result is the back face with
  upside down image.  With an upside down projection, it gives the wanted result.

## SurfaceComposerClient

- There are 4 types of CBLK: `surface_flinger_cblk_t`, `display_cblk_t`,
  `per_client_cblk_t`, and `layer_cblk_t`
- `lockSurface` locks and returns the info of the back buffer.  Current dirty
  region is copied from front buffer to back buffer, and is updated to new one.
- `unlockAndPostSurface` sends current dirty region to server (by storing in
  `layer_cblk_t`), calls `unlock_layer_and_post` and signal server.  If there is
  swap rectangle, it is sent instead of dirty region.
- The rest deals with states, which is sent to server through `setState`

## Surface

- For 2D usage, `Surface` wraps the underlying buffer in a `SkBitmap` and
  operates on a `SkCanvas`.  See `Surface_lockCanvas`.
- For 3D usage, `Surface` wraps the underlying buffer in a
  `EGLNativeWindowSurface` and operates in a `GL` context.  The
  `EGLNativeWindowSurface` is passed to `eglCreateWindowSurface` and the
  returned `EGLSurface` is made current in `EGL`.  See `eglCreateWindowSurface`
  in `EGLImpl.java`.

## SurfaceFlinger objects

- `LayerBase` implements `ISurface`
- `Layer` is a `LayerBase` with `SurfaceTextureLayer`
- `SurfaceTexture` (it may also be used by a client)
  - `mQueue` is a queue of indices of the buffers that are to be displayed.  In
    sync mode, every buffer will be displayed in order.  In async mode, the
    queue has at most one index.  When a new buffer is queued before the current
    one is displayed, the new one replaces the current one.
  - A buffer is initially `FREE`.  After calling `dequeueBuffer`, it becomes
    `DEQUEUED`.  After calling `enqueueBuffer`, it becomes `QUEUED`.  When
    `updateTexImage` is called, the buffer is made current (streamed to the
    texture object) and is erased from `mQueue`.  However, it remains `QUEUED`
    until next `updateTexImage` call.  This makes sure the client does not see
    the buffer that is still in use.
    - In sync mode, a current buffer can be dequeued when it is free.  But that
      is never the case.  The doc is outdated?
  - Sync mode: `dequeueBuffer` blocks until there is a slot available.  Buffers
    queued in sync mode will be retired in order.
    - That is, every frame is shown.
  - Async mode: `dequeueBuffer` never blocks.  It returns error if there is no
    slot available.  There is at most one queued buffer and queueing another
    buffer will replace the current one.
    - That is, there can be frame drops
  - `updateTexImage` streams the first buffer in the queue to the texture object
    and clears it from the queue.  The state of the buffer is still `QUEUED`
    until next `updateTexImage` call
  - A video decoder may process multiple frames parellelly.  It tends to dequeue
    multiple buffers at the same time.
  - `MIN_ASYNC_BUFFER_SLOTS` is 3
  - `MIN_SYNC_BUFFER_SLOTS` is 2
  - client buffer count is used only when it is not zero.
  - at most one buffer can be dequeued if client buffer count is not set

## `captureScreen`

- `renderScreenImplLocked` draws all layers from bottom to top
  - `layersSortedByZ` is from bottom to top
  - when an activity has a surface view, the layer of the surface view is
    under the layer of the activity.  A hole is dug out from the layer of the
    activity
    - `transparentRegionHint` is just a hint
    - I think it clears the alpha channel to 0.0

## Client Composition

- U defaults to `SKIA_GL_THREADED`
  - `debug.renderengine.backend` can override

## VSync

- `SurfaceFlinger::onComposerHalVsync` is the hw vsync callback
  - `HWComposer::onVsync` validates and maps the vsync to display
  - `Scheduler::addResyncSample` feeds the vsync to the scheduler
    - `getVsyncSchedule` returns the `VsyncSchedule` for the display
    - `VsyncSchedule::addResyncSample` calls `VSyncReactor::addHwVsyncTimestamp`
      - `addVsyncTimestampLocked` calls `VSyncPredictor::addVsyncTimestamp`
        - `VSyncPredictor` predicts future vsyncs even when hw vsyncs are
          disabled to save power
  - if refresh rate changes, `Scheduler::modulateVsync` is called with
    `VsyncModulator::onRefreshRateChangeCompleted`
- `Scheduler::registerDisplayInternal` calls `promotePacesetterDisplay`
  - `applyNewVsyncSchedule`
    - `MessageQueue::onNewVsyncSchedule` registers `MessageQueue::vsyncCallback`
    - `EventThread::onNewVsyncSchedule` registers `EventThread::onVsync`
    - `vsyncSchedule->getDispatch` is `VSyncDispatchTimerQueue`
      - `mTimeKeeper` is a `scheduler::Timer` which uses `timerfd`
- `MessageQueue::vsyncCallback` is called on `VSYNC-sf` when timer alarms
  - `mVsync` toggles `VSYNC-sf` counter
  - `MessageQueue::Handler::dispatchFrame` sends a msg
  - it wakes up the main thread `SurfaceFlinger::run -> Schedule::run -> MessageQueue::waitMessage`
    - `MessageQueue::Handler::handleMessage` calls `Scheduler::onFrameSignal`
- `EventThread::onVsync` is called on `VSYNC-app` when timer alarms
  - `mVsyncTracer` toggles `VSYNC-app` counter
  - it pushes a `DISPLAY_EVENT_VSYNC` event to `mPendingEvents`
  - it wakes up `EventThread::threadMain` to call `dispatchEvent` to dispatch
    the event to apps
- `adb shell dumpsys SurfaceFlinger --vsync`
  - `VSYNC-sf` and `VSYNC-app` are `workDuration` before the next vsync
  - `earlyGpu` is used for gpu composition
    - it has the largest `workDuration`
  - `early` is used for hw composition during state change
    - it has the medium `workDuration` to reduce jank
  - `late` is used for hw composition during static state
    - it has the smallest `workDuration` to reduce input latency
- `MessageQueue::scheduleFrame` arms the `VSYNC-sf` alarm
  - `VSyncCallbackRegistration::schedule -> VSyncDispatchTimerQueue::schedule
    -> VSyncDispatchTimerQueueEntry::schedule`
  - it determines `nextVsyncTime` and works out the rest
    - `nextReadyTime = nextVsyncTime - timing.readyDuration`
    - `nextWakeupTime = nextReadyTime - timing.workDuration`
    - `timing.readyDuration` is typically 0
  - it calls `MessageQueue::vsyncCallback` with the expected timeline
    - wakeup/start time, which is when sf should wake up
    - ready/end time, which is when sf should present
    - vsync/present time, which is when the frame shows up on screen
  - as for actual timeline
    - `SurfaceFlinger::commit` calls `setSfWakeUp`
    - `SurfaceFlinger::onCompositionPresented` calls `setSfPresent`

## Perfetto

- from `VSYNC-sf` to gpu completion
  - `Timer::threadMain` calls `VSyncDispatchTimerQueue::timerCallback`
    - `MessageQueue::vsyncCallback` calls `MessageQueue::Handler::dispatchFrame`
  - `SurfaceFlinger::run` calls `MessageQueue::Handler::handleMessage`
    - `Scheduler::onFrameSignal`
      - `FrameTargeter::beginFrame`
      - `SurfaceFlinger::commit` applies pending transactions, latches
        buffers, updates layer hierarchy, etc.
      - `SurfaceFlinger::composite` calls `CompositionEngine::present` to
        composite and present a frame
        - `Output::prepareFrame`
          - `Display::chooseCompositionStrategy` chooses gpu and/or hwc
          - `Display::applyCompositionStrategy` applies the strategy
        - if gpu,
          - `Output::dequeueRenderBuffer` dequeues
          - `Output::composeSurfaces` calls `RenderEngineThreaded::drawLayers`
          - `RenderSurface::queueBuffer` queues
            - `Surface::queueBuffers` traces `GPU completion`
            - `LegacyFramebufferSurface::advanceFrame` sets the buffer as
              hwc client target
        - `Display::presentFrame` presents to hwc
  - `RenderEngineThreaded::threadMain` calls the `drawLayers` lambda
    - `RenderEngine::updateProtectedContext`
    - `SkiaRenderEngine::drawLayersInternal`
      - `trace` traces `RE Completion`
