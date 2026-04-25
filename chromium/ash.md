# Chromium ash

## Ash: Notification Center

- cap lock shows an notification in the system tray
- trace view
  - browser `evdev` thread `EventConverterEvdevImpl::OnFileCanReadWithoutBlocking`
    - `ProxyDeviceEventDispatcher::DispatchKeyEvent` dispatches the event to
      the main thread
  - browser `CrBrowserMain` thread `EventFactoryEvdev::DispatchKeyEvent`
    - `PlatformEventSource::DispatchEvent`
    - `DrmWindowHost::DispatchEvent`
    - `PlatformWindowDelegate::DispatchEvent`
    - `AshWindowTreeHostPlatform::DispatchEvent`
    - `WindowTreeHostPlatform::DispatchEvent`
    - `EventSource::SendEventToSink`
    - the event somewhat triggers
      `CapsLockNotificationController::OnCapsLockChanged`
      - `ash::CreateSystemNotificationPtr` creates a notification
      - `MessageCenterImpl::AddNotification` adds the notification
    - `NotificationCenterController::OnNotificationAdded` creates a view for
      the notification
    - `LayerTreeHost::DoUpdateLayers` updates layers
    - `LayerTreeHostImpl::FinishCommit` commits the layers
    - `LayerTreeHostImpl::CommitComplete` calls `TileManager::PrepareTiles`
  - browser `CompositorTileWorker1` thread `RasterTaskImpl::RunOnWorkerThread`
    - `OneCopyRasterBufferProvider::RasterBufferImpl::Playback`
      - this is software rasterization
  - browser `CrBrowserMain` thread `SingleThreadProxy::DoComposite`
    - `LayerTreeHostImpl::PrepareToDraw`
    - `LayerTreeHostImpl::DrawLayers`
  - gpu `VizCompositorThread` thread
    - `Surface::CommitFrame`
    - `DisplayScheduler::OnBeginFrame`
    - `DisplayScheduler::OnBeginFrameDeadline`
      - `DisplayScheduler::DrawAndSwap`
      - `Display::DrawAndSwap`
        - `DirectRenderer::DrawFrame`
          - `SkiaRenderer::BeginDrawingFrame`
          - `DirectRenderer::DrawRenderPass` and `SkiaRenderer::FinishDrawingRenderPass`
            - `SkiaRenderer::FlushOutputSurface` calls `SkiaOutputSurfaceImpl::FlushGpuTasks`
          - `SkiaRenderer::FinishDrawingFrame`
        - `SkiaRenderer::SwapBuffers`
          - `SkiaOutputSurfaceImpl::SwapBuffers`
          - `SkiaOutputSurfaceImpl::Flush`
  - gpu `CrGpuMain` thread
    - `SkiaOutputSurfaceImplOnGpu::FinishPaintRenderPass`
    - `SkiaOutputSurfaceImplOnGpu::SwapBuffers`
- when gpu rasterization,
  `GpuRasterBufferProvider::RasterBufferImpl::Playback` is called instead of
  `OneCopyRasterBufferProvider::RasterBufferImpl::Playback`
  - `GpuRasterBufferProvider::RasterBufferImpl::RasterizeSource` calls
    `BeginRasterCHROMIUM`, `RasterCHROMIUM`, and `EndRasterCHROMIUM`
