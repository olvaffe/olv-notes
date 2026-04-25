# Chromium components/exo

## GPU and EXO

- exo initialization
  - from `BrowserMainLoop` of the browser process
    - `ChromeContentBrowserClient::CreateBrowserMainParts` creates
      `ChromeBrowserMainExtraPartsAsh`
    - `ChromeBrowserMainExtraPartsAsh::PreProfileInit` calls
      `ExoParts::CreateIfNecessary`
    - `ExoParts::ExoParts` calls `WaylandServerController::CreateIfNecessary`
    - `WaylandServerController::WaylandServerController` calls
      `wayland::Server::Create`
- `zwp_linux_dmabuf_v1`
  - init
    - `wayland::Server::Initialize` creates `WaylandDmabufFeedbackManager`
    - `WaylandDmabufFeedbackManager::WaylandDmabufFeedbackManager` initializes
      based on `gpu::Capabilities`
      - `BrowserMainLoop::PostCreateThreadsImpl` creates a
        `VizProcessTransportFactory` and calls
        `aura::Env::set_context_factory`
      - `VizProcessTransportFactory::TryCreateContextsForGpuCompositing`
        creates a `viz::ContextProviderCommandBuffer`
      - `ContextProviderCommandBuffer::BindToCurrentSequence` sets `impl_` to
        a `gpu::gles2::GLES2Implementation` wrapping a
        `gpu::gles2::GLES2CmdHelper`
      - `ImplementationBase::ImplementationBase` calls
        `gpu_control->GetCapabilities` to initialize caps
        - `BrowserGpuChannelHostFactory::EstablishGpuChannelSync` creates a
          `GpuChannelHost`
        - `gpu::CommandBufferProxyImpl` is a `GpuControl`
- formats
  - wayland protocol
    - `wl_shm.format` uses fourcc
    - `zwp_linux_dmabuf_v1` uses fourcc as well
  - exo
    - `wl_shm` uses `shm_supported_formats` to map fourcc to
      `gfx::BufferFormat`
    - `zwp_linux_dmabuf_v1` uses `ui::GetBufferFormatFromFourCCFormat` to map
      fourcc to `gfx::BufferFormat`
  - when exo creates a `ClientSharedImage`,
    - if allocating, it specifies `viz::SharedImageFormat` directly
    - if importing from `gfx::GpuMemoryBuffer`, no format is specified
      - the `viz::SharedImageFormat` of the image will be drived from
        `gfx::BufferFormat`
- `CreateSharedImage`
  - exo uses `ClientSharedImageInterface::CreateSharedImage`
    - if importing, the variant with `gfx::GpuMemoryBuffer`
  - `SharedImageInterfaceProxy::CreateSharedImage`
  - `SharedImageStub::OnCreateGMBSharedImage`
  - `SharedImageStub::CreateSharedImage`
    - if importing, the variant with `gfx::GpuMemoryBufferHandle` and
      `gfx::BufferFormat`
  - `SharedImageFactory::CreateSharedImage`
  - `OzoneImageBackingFactory::CreateSharedImage`
    - if importing,
      - `GbmSurfaceFactory::CreateNativePixmapFromHandle`
      - `DrmThreadProxy::CreateBufferFromHandle`
      - `DrmThread::CreateBufferFromHandle`
      - `GbmDevice::CreateBufferFromHandle`
      - `gbm_bo_import`
      - the `viz::SharedImageFormat` of the image is derived from
        `gfx::BufferFormat`
        - `GetPlaneBufferFormat` returns the plane format in
          `gfx::BufferFormat`
        - `viz::GetSinglePlaneSharedImageFormat` returns the image format
- composition
  - when viz is ready to composite the image, it calls
    `SkiaOutputSurfaceImpl::MakePromiseSkImage` to create a `SkImage`
    - for rgba, it uses `MakePromiseSkImageSinglePlane`
- format example
  - when the wayland client uses `DRM_FORMAT_XRGB8888`
  - exo calls `GetBufferFormatFromFourCCFormat` to use `gfx::BufferFormat::BGRX_8888`
  - `OzoneImageBackingFactory` calls `GetSinglePlaneSharedImageFormat` to use
    `SinglePlaneFormat::kBGRX_8888`
  - viz calls `ToClosestSkColorType` to use `kRGB_888x_SkColorType`
  - viz also calls `GetGrBackendFormatForTexture` to use `VK_FORMAT_B8G8R8A8_UNORM`
    - if vk, `GetGrBackendFormatForTexture` calls `gpu::ToVkFormat` that calls
      `ToVkFormatSinglePlanar` that calls `ToVkFormatSinglePlanarInternal`
  - note that viz picks both `kRGB_888x_SkColorType` and
    `VK_FORMAT_B8G8R8A8_UNORM`
    - inside skia, `GrVkCaps::initFormatTable` maps `VK_FORMAT_B8G8R8A8_UNORM`
      to `GrColorType::kBGRA_8888` which results in a conflict

## Compositor

- compositor with a running vkcube in crosvm
  - some time after vsync, viz's `DelayBasedBeginFrameSource::OnTimerTick` is
    called.  The function calls `OnBeginFrame` on all observers
    - ash?'s `ExternalBeginFrameSource::OnBeginFrame` is called.  This
      prepares a frame for the compositor and ends with `Surface::Commit`
      which does `SubmitCompositorFrame`
  - after a bit of wait, viz calls `DisplayScheduler::OnBeginFrameDeadline`
    - this calls `DisplayScheduler::DrawAndSwap` which ultimately calls
      `DirectRenderer::DrawFrame` and `SkiaRenderer::SwapBuffers` using the
      frames submitted by the clients (I think it uses frames submitted by
      clients in the last vsync; clients need a bit time until they
      `SubmitCompositorFrame`)
    - `SkiaRenderer::SwapBuffers` wakes up the gpu process main thread.  The
      gpu proces main thread, among other things, calls
      `SkiaOutputSurfaceImplOnGpu::SwapBuffers` which calls
      `GbmSurfaceless::SwapBuffersAsync`
    - `GbmSurfaceless::SwapBuffersAsync` wakes up `DrmThread`.  `DrmThread`
      calls `DrmThread::SchedulePageFlip` and `DrmThread::OnPlanesReadyForPageFlip`
- EXO and the wayland server is a part of ash

## Wayland

- XDG_RUNTIME_DIR=/var/run/chrome
