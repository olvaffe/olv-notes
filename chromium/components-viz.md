Chromium components/viz
=======================

## GPU and viz

- <https://chromium.googlesource.com/chromium/src.git/+/main/components/viz/README.md>
- `GpuServiceImpl`
  - it is created from `VizMainImpl::CreateGpuService`
  - the ctor creates a `GpuMemoryBufferFactory`
  - `GpuServiceImpl::InitializeWithHost` creates a `GpuChannelManager`
- `GpuMemoryBufferFactory` is the GMB allocator
  - `gpu::GpuMemoryBufferFactory::CreateNativeType` creates
    `GpuMemoryBufferFactoryNativePixmap`
  - `GpuMemoryBufferFactory::CreateGpuMemoryBuffer` calls ozone's
    `SurfaceFactoryOzone::CreateNativePixmap` and the returned
    `gfx::GpuMemoryBufferHandle` wraps `gfx::NativePixmap`
- `GpuChannelManager` is the gpu context
  - `GetSharedContextState` creates the gpu context and initializes skia
  - `gl::init::CreateGLContext` creates the GL context and makes it current
  - `SharedContextState::InitializeSkia` initializes skia
- life of a pixel
  - chrome comopsitor (cc) in the renderer processes generates
    `CompositorFrame`s and calls `SubmitCompositorFrame` to submit them to viz
  - `Display::DrawAndSwap`
    - calls `SurfaceAggregator::Aggregate` to aggregate `CompositorFrame` into
      a `AggregatedFrame`
    - calls `DirectRenderer::DrawFrame` to draw the aggregated frame
    - calls `SkiaRenderer::SwapBuffers` to swap
    - I think this can also promote some frames to hw overlays and skip gpu
      composition for them
  - `display_embedder` contains platform-specific code for display
- I have a trace for chrome playing a video
  - it appears that `Chrome_ChildIOThread` is the io thread
  - renderer: `Chrome_ChildIOT` wakes up (timer-based?)
  - renderer: `VideoFrameCompo` wakes up (sw-decode a frame?)
  - renderer: `Chrome_ChildIOT` wakes up (submits frame to gpu process?)
  - gpu: `Chrome_ChildIOThread` wakes up and calls `FlushDeferredRequests`
    - gpu: `CrGpuMain` calls `ExecuteDeferredRequest` in response
  - renderer: `Chrome_ChildIOT` wakes up (submission acked?)
  - renderer: `VideoFrameCompo` wakes up (more work submitted?)
  - gpu: `Chrome_ChildIOThread` wakes up
  - gpu: `VizCompositorThread` wakes up to `Surface::CommitFrame`,
    `DisplayScheduler::DrawAndSwap`, and `SkiaRenderer::SwapBuffers`
  - gpu: `CrGpuMain` wakes up to `SkiaOutputSurfaceImplOnGpu::SwapBuffers`
  - gpu: `DrmThread` wakes up to `DrmThread::SchedulePageFlip` and
    `DrmThread::OnPlanesReadyForPageFlip`
    - the kernel creates a crtc fence that is signaled on next vblank
  - gpu: `DrmThread` wakes up to `OnDrmEvent` (after the crtc fence signals)
  - gpu: `CrGpuMain` wakes up
    - this probably marks the previous buffer available
  - gpu: `VizCompositorThread` wakes up
- in this video playback trace,
  - there is a new frame every ~33ms
    - this is likely the fps of the video stream
    - a flip is scheduled every other vsync
  - sometimes there is an extra frame update
    - this is likely from non-video frame change
    - an extra flip is scheduled
  - pseudo events
    - `DisplayScheduler:pending_swaps` starts slightly before
      `SkiaRenderer::SwapBuffers` and ends slightly after
      `DrmThread::OnPlanesReadyForPageFlip`
    - `Graphics.Pipeline.DrawAndSwap`
      - start early in `DisplayScheduler::DrawAndSwap`
      - `draw` phase corresponds to `DirectRenderer::DrawFrame`
      - `WaitForSwap` phase starts before `SkiaRenderer::SwapBuffers` and ends
        in `SkiaOutputSurfaceImplOnGpu::SwapBuffers`
      - `Swap` phase starts in `SkiaOutputSurfaceImplOnGpu::SwapBuffers` and
        ends after `DrmThread::OnPlanesReadyForPageFlip`
      - `WaitForPresentation` phase starts after
        `DrmThread::OnPlanesReadyForPageFlip` and ends in `OnDrmEvent`
- I have a trace for chrome showing camera preview
  - when a capture completes
    - with capture time of 33ms and a queue of 3 captures, a capture completes
      99ms later after scheduled
  - `cros_camera_service`: `Capture request` wakes up to
    - `CameraClient::RequestHandler::WriteStreamBuffers` 
    - `CameraDeviceAdapter::Notify`
      - this wakes up `DefaultCaptureR` which wakes up `EffectsGlThread`
    - `CameraDeviceAdapter::ProcessCaptureResult`
      - this wakes up `DefaultCaptureR` again which wakes up
        `EffectsGlThread`, `CameraCallbackO`, and `FenceSyncThread`
      - this also wakes up `CameraMonitor`
  - main: `Chrome_ChildIOThread` and `CameraDeviceIpcThread0` wake up to
    - schedule another capture
      - `cros_camera_service`: `MojoIpcThread` wakes up
      - `cros_camera_service`: `CameraDeviceOps` wakes up to
        `CameraDeviceAdapter::ProcessCaptureRequest`
    - notify renderer about the current capture
      - the rest is similar to the video playback trace above
- I have a trace for a key press
  - a `InputLatency::Char` meansures the latency of a key press to page flip
    completion
  - key press
  - main: `evdev` wakes up
  - main: `CrBrowserMain` wakes up to call
    `RenderWidgetHostViewBase::OnKeyEvent` and
    `RenderWidgetHostImpl::ForwardKeyboardEvent`
    - a `LatencyInfo` is created to track the latency of the input event
    - `STEP_SEND_INPUT_EVENT_UI`
  - renderer: `Compositor` wakes up to call `WidgetInputHandlerImpl::DispatchEvent`
    - `STEP_HANDLE_INPUT_EVENT_IMPL`
    - `STEP_DID_HANDLE_INPUT_AND_OVERSCROLL`
  - renderer: `CrRendererMain` wakes up to call
    `WidgetBaseInputHandler::OnHandleInputEvent`
    - `STEP_HANDLE_INPUT_EVENT_MAIN`
    - `STEP_HANDLED_INPUT_EVENT_MAIN_OR_IMPL`
    - `STEP_HANDLED_INPUT_EVENT_IMPL`
  - renderer: `Compositor` wakes up to call `ProxyImpl::ScheduledActionDraw`
    and `MainFrame.Draw`
    - `STEP_SWAP_BUFFERS`
  - gpu: `VizCompositorThread` wakes up to call `Display::DrawAndSwap`
    - `STEP_DRAW_AND_SWAP`
  - gpu: `CrGpuMain` wakes up after `OnDrmEvent` (after page flip)
- `PipelineReporter` track appears to be the most important info about frames
  - start: hw vsync
  - end: hw flip complete of the following frame
    - that is, the scanout buffer is ready for reuse
  - it is usually 2 vsyncs long
    - After a hw vsync, viz asks the renderer to begin a frame
    - The renderer spends a few ms to composite the frame and submits it to viz
    - viz spends a few ms to render the frame and submits it to kernel for
      page flip
    - on next hw flip, the frame becomes front
    - on yet another hw flip, the frame becomes available for reuse
  - <https://android.googlesource.com/platform/external/perfetto/+/refs/heads/main/protos/perfetto/trace/track_event/chrome_frame_reporter.proto>
    - state
      - `STATE_NO_UPDATE_DESIRED` no real change
      - `STATE_PRESENTED_ALL` all changes are presented
      - `STATE_PRESENTED_PARTIAL` some changes are presented
      - `STATE_DROPPED` some required changes are missing
    - frame drop reason
      - `REASON_DISPLAY_COMPOSITOR` viz decides to drop
      - `REASON_MAIN_THREAD` the main thread decides to drop
      - `REASON_CLIENT_COMPOSITOR` the cc thread decides to drop
- threads
  - `CreateAndStartIOThread` creates `GpuIOThread` thread
  - `VizCompositorThreadRunnerImpl` creates `VizCompositorThread` thread
  - `CompositorGpuThread::Create` creates `CompositorGpuThread` thread

## Perfetto: Graphics Pipeline

- `STEP_ISSUE_BEGIN_FRAME` is on gpu `VizCompositorThread`
  - `RootCompositorFrameSinkImpl` has a `DelayBasedBeginFrameSource` as
    `synthetic_begin_frame_source_` that ticks at vsync freq
  - on `DelayBasedBeginFrameSource::OnTimerTick`,
    - `CompositorFrameSinkSupport::OnBeginFrame` emits the trace event
    - `DisplayScheduler::BeginFrame` calls
      `DisplayScheduler::ScheduleBeginFrameDeadline` to schedule a deadline
      - upon deadline, viz latches and aggregates all client frames, draws,
        and presents
      - it is picked to be next vsync minus
        `BeginFrameArgs::DefaultEstimatedDisplayDrawTime`, to give clients
        time to generate client frames and to give viz time to aggregate, draw
        and swap
- `STEP_RECEIVE_BEGIN_FRAME` is on renderer `Compositor` (the impl thread)
  - `AsyncLayerTreeFrameSink::OnBeginFrame` emits the trace event
  - `ExternalBeginFrameSource::OnBeginFrame` calls
    `Scheduler::ScheduleBeginImplFrameDeadline` to schedule a deadline on
    `Compositor` thread
    - the idea is to reduce latency
  - upon deadline, `Scheduler::OnBeginImplFrameDeadline` is called
    - `ProxyImpl::ScheduledActionSendBeginMainFrame` tells `CrRendererMain`
      (the main thread) to update and commit the layer tree
    - `ProxyImpl::ScheduledActionDraw` trigger the following steps
- `STEP_GENERATE_COMPOSITOR_FRAME` and `STEP_SUBMIT_COMPOSITOR_FRAME` are on
  renderer `Compositor`
  - `ProxyImpl::DrawInternal` emits `STEP_GENERATE_COMPOSITOR_FRAME`
    - `LayerTreeHostImpl::DrawLayers` emits `STEP_SUBMIT_COMPOSITOR_FRAME`
- `STEP_RECEIVE_COMPOSITOR_FRAME` is on gpu `VizCompositorThread`
  - `CompositorFrameSinkSupport::MaybeSubmitCompositorFrame` emits the trace
    event
  - `Surface::QueueFrame` calls `Surface::CommitFrame`
    - this may call `DisplayScheduler::OnPendingSurfacesChanged` and
      re-schedule the deadline to happen immediately if all surfaces are
      activated
- `STEP_DRAW_AND_SWAP`, `STEP_SURFACE_AGGREGATION`, and
  `STEP_SEND_BUFFER_SWAP` are on gpu `VizCompositorThread`
  - `DisplayScheduler::OnBeginFrameDeadline` calls `Display::DrawAndSwap`
  - `Display::DrawAndSwap` emits `STEP_DRAW_AND_SWAP` and
    `STEP_SEND_BUFFER_SWAP`
    - `SurfaceAggregator::Aggregate` emits `STEP_SURFACE_AGGREGATION`
    - `SkiaRenderer::SwapBuffers` triggers the following step
- `STEP_BUFFER_SWAP_POST_SUBMIT` is on gpu `CrGpuMain`
  - `SkiaOutputSurfaceImplOnGpu::SwapBuffers` submits the gpu job and calls
    `SkiaOutputSurfaceImplOnGpu::PostSubmit` which emits the trace event
- on wayland, `STEP_BACKEND_SEND_BUFFER_SWAP`,
  `STEP_BACKEND_SEND_BUFFER_POST_SUBMIT`, and
  `STEP_BACKEND_FINISH_BUFFER_SWAP` are on browser `CrBrowserMain`
  - `WaylandFrameManager::RecordFrame` emits `STEP_BACKEND_SEND_BUFFER_SWAP`
  - `WaylandFrameManager::PlayBackFrame` emits `STEP_BACKEND_SEND_BUFFER_POST_SUBMIT`
  - `WaylandFrameManager::MaybeProcessSubmittedFrames` emits
    `STEP_BACKEND_FINISH_BUFFER_SWAP`
- `STEP_FINISH_BUFFER_SWAP` is on gpu `CrGpuMain`
  - `SkiaOutputDevice::FinishSwapBuffers` emits the trace event when the swap
    is scheduled
    - the swap will happen on next vsync
- `STEP_SWAP_BUFFERS_ACK` is on gpu `VizCompositorThread`
  - `SkiaOutputSurfaceImpl::DidSwapBuffersComplete` calls
    `Display::DidReceiveSwapBuffersAck` which emits the trace event
