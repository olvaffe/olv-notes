Chromium ui/ozone
=================

## GPU and Ozone

- `SurfaceFactoryOzone::GetGLOzone`
  - on wayland, `GLOzoneEGLWayland` is returned
  - on cros, `GLOzoneEGLGbm` is returned
- `SurfaceFactoryOzone::CreateVulkanImplementation`
  - on wayland, `VulkanImplementationWayland` is returned
  - on cros, `VulkanImplementationGbm` is returned
- `SurfaceFactoryOzone::CreateOffscreenGLSurface`
  - with `EGL_KHR_surfaceless_context`, this creates nothing
  - this is called from `gpu::CollectContextGraphicsInfo` and is used for
    `GpuInit::default_offscreen_surface_`
- `SurfaceFactoryOzone::CreateSurfacelessViewGLSurface`
  - this creates almost nothing and is used when we want to render to
    `NativePixmap`
  - this is called from `SkiaOutputSurfaceDependencyImpl::CreatePresenter`
- `SurfaceFactoryOzone::CreateNativePixmap`
  - this allocates a `NativePixmap`, which is a `gbm_bo`
  - example call stack
    - `ui::WaylandSurfaceFactory::CreateNativePixmap()`
    - `gpu::OzoneImageBackingFactory::CreateSharedImageInternal()`
    - `gpu::OzoneImageBackingFactory::CreateSharedImage()`
    - `gpu::SharedImageFactory::CreateSharedImage()`
    - `viz::OutputPresenter::Image::Initialize()`
    - `viz::OutputPresenterGL::AllocateImages()`
    - `viz::SkiaOutputDeviceBufferQueue::RecreateImages()`
    - `viz::SkiaOutputDeviceBufferQueue::Reshape()`
    - `viz::SkiaOutputSurfaceImplOnGpu::Reshape()`
    - ...
    - `viz::SkiaOutputSurfaceImpl::FlushGpuTasksWithImpl()::$_0::operator()()`
  - other places such as
    `viz::SkiaOutputSurfaceImplOnGpu::CreateSolidColorSharedImage` also call
    `gpu::SharedImageFactory::CreateSharedImage`
- `SurfaceFactoryOzone::ImportNativePixmap`
  - this imports a `NativePixmap`, which is a `gbm_bo`, as a GL texture and
    returns `NativePixmapGLBinding`
  - example call stack
    - `ui::(anonymous namespace)::GLOzoneEGLWayland::ImportNativePixmap()`
    - `gpu::GLOzoneImageRepresentationShared::GetBinding()`
    - `gpu::GLOzoneImageRepresentationShared::CreateTextureHolderPassthrough()`
    - `gpu::GLOzoneImageRepresentationShared::CreateShared()`
    - `gpu::GLTexturePassthroughOzoneImageRepresentation::Create()`
    - `gpu::OzoneImageBacking::ProduceGLTexturePassthrough()`
    - `gpu::OzoneImageBacking::ProduceSkiaGanesh()`
    - `gpu::SharedImageBacking::ProduceSkia()`
    - `gpu::SharedImageManager::ProduceSkia()`
    - `gpu::SharedImageRepresentationFactory::ProduceSkia()`
    - `viz::OutputPresenter::Image::Initialize()`
    - `viz::OutputPresenterGL::AllocateImages()`
  - other places such as `OzoneImageBacking::UploadFromMemory` call
    `ProduceSkiaGanesh` which also import the native pixmap

## CrOS DRM Overlays

- on startup,
  - there is an overlay for fullscreen XR24 fb
    - this is gpu-composited
  - if the cursor is on screen, there is another overlay for cursor AR24 fb
- creating chrome windows does not change the overlay strategy
  - whether the windows are maximized or fullscreen
- video playback
  - there can be 2 overlays for video-sized NV12 fb
    - NV12 is planar and requires 2 overlays
    - hw scaling is used for video resizing
  - there is another overlay for fullscreen XR24 or AR24 fb
    - this is gpu-composited
    - XR24 if nothing is on top of the video
    - AR24 if anything (e.g., video controls, captions, etc.) is on top of the video
  - when the video is paused, the strategy might or might not change
    - it could choose to gpu-comopsite everything
- arcvm app
  - there can be an overlay for the arc app
    - the format can be AR24, AB24, XB24, etc.
  - there is another overlay for fullscreen AR24 fb
    - this is gpu-composited
    - this includes arc app client decoration
  - sometimes the compositor might choose to gpu-composite the arc app as well
- arcvm SurfaceView
  - other than an overlay for the app, there can be yet another overlay for
    the surfaceview
  - cros may choose different strategies
    - it may gpu-composite everything
    - it may convert the format (`RB16->AB24`) of the surfaceview while still
      using an overlay for it
      - this is tinted if `#tint-composited-content` is `Enabled`
      - this does not happen if `#overlay-strategies` is `None`

## DRM

- monitor hotplug
  - `BrowserMain` creates and initializes `BrowserMainRunnerImpl`
    - `BrowserMainLoop::Init` calls
      `ChromeContentBrowserClient::CreateBrowserMainParts`
      - `ChromeBrowserMainExtraPartsAsh` calls `ash::Shell::CreateInstance`
        indirectly
      - `ash::Shell::InitializeDisplayManager` creates a
        `DrmNativeDisplayDelegate`
    - `BrowserMainLoop::InitializeToolkit` calls `aura::Env::CreateInstance`
      - `aura::Env::Init` calls `ui::OzonePlatform::InitializeForUI`
      - `OzonePlatformDrm::InitializeUI` creates
        - `DeviceManager` to monitor udev devs
        - `DrmWindowHostManager` to manage drm fbs?
        - `DrmDisplayHostManager` to manage drm devs and crtcs
      - `DrmDisplayHostManager::OnDeviceEvent` handles udev events
  - on monitor hotplug,
    - browser `DrmDisplayHostManager::OnUpdateGraphicsDevice` calls gpu
      `HostDrmDevice::GpuShouldDisplayEventTriggerConfiguration`
    - gpu process
      - viz `HostDrmDevice::GpuShouldDisplayEventTriggerConfiguration`
      - drm `DrmThread::ShouldDisplayEventTriggerConfiguration`
        - `DrmGpuDisplayManager::ShouldDisplayEventTriggerConfiguration` parses
          the event to determine if it should trigger
    - browser
      `DrmDisplayHostManager::GpuShouldDisplayEventTriggerConfiguration` calls
      `DrmDisplayHostManager::NotifyDisplayDelegate`
    - `DrmNativeDisplayDelegate::OnConfigurationChanged` calls
      `DisplayConfigurator::OnConfigurationChanged`
    - `DisplayConfigurator::ConfigureDisplays` configures displays
      - `UpdateDisplayConfigurationTask::Run` calls
        - `DrmNativeDisplayDelegate::GetDisplays`
        - `DrmDisplayHostManager::UpdateDisplays`
        - `HostDrmDevice::GpuRefreshNativeDisplays`
        - `DrmThread::RefreshNativeDisplays`
        - `DrmGpuDisplayManager::GetDisplays`
        - `UpdateDisplayConfigurationTask::OnDisplaysUpdated`
      - `UpdateDisplayConfigurationTask::EnterState` calls
        - `DisplayConfigurator::DisplayLayoutManagerImpl::GetDisplayLayout`
        - `ConfigureDisplaysTask::Run`
        - `DrmNativeDisplayDelegate::Configure`
        - `DrmDisplayHostManager::ConfigureDisplays`
        - `HostDrmDevice::GpuConfigureNativeDisplays`
        - `DrmThread::ConfigureNativeDisplays`
        - `DrmGpuDisplayManager::ConfigureDisplays`
        - `UpdateDisplayConfigurationTask::OnStateEntered`
    - `DisplayConfigurator::OnConfigured` is called after displays are
      configured
      - `DisplayConfigurator::NotifyDisplayStateObservers`
      - `DisplayChangeObserver::OnDisplayConfigurationChanged`
      - `DisplayManager::OnNativeDisplaysChanged`
      - `DisplayManager::UpdateDisplaysWith`
      - `DisplayManager::NotifyDisplayAdded`
        - `WindowTreeHostManager::CreateDisplay` creates a
          `AshWindowTreeHost` for the display
      - more
    - when mirroring
      - `WindowTreeHostManager::AddWindowTreeHostForDisplay` creates a
        `AshWindowTreeHost` for the display with `MirrorWindowController`
      - `MirrorWindowController::UpdateWindow` calls
        `CreateRootWindowTransformerForMirroredDisplay` to create a
        transformer
        - this scales the window when the two displays have different
          resolution
- gpu process `DrmThread`
  - `OzonePlatformDrm::InitializeGPU` creates a `DrmThreadProxy` and
    `OzonePlatformDrm::AfterSandboxEntry` calls
    `DrmThreadProxy::StartDrmThread` to create the thread
  - `DrmDeviceManager` manages drm devices
    - `DrmThread::AddGraphicsDevice` adds a drm device to the manager
    - each drm device is represented by a `DrmDevice`
  - `DrmGpuDisplayManager` manages drm crtcs
    - `DrmGpuDisplayManager::GetDisplays` queries crtcs and creates
      `DrmDisplay` to manage them
  - `ScreenManager` manages active drm crtcs
    - `ScreenManager::AddDisplayControllersForDisplay` adds a
      `HardwareDisplayController` for a `DrmDisplay`
- gpu process `VizCompositorThread`
  - when a monitor is added/removed, `DrmDisplayHostManager::OnDeviceEvent`

## Protected Contents

- during gpu initialization, `ChromeContentGpuClient::GpuServiceInitialized`
  registers `arc::ProtectedBufferManager::GetProtectedNativePixmapFor`
- when `gpu::OzoneImageBackingFactory::CreateSharedImage` imports a
  `gfx::GpuMemoryBufferHandle`, it calls
  `ui::GbmSurfaceFactory::CreateNativePixmapFromHandle` which calls
  `arc::ProtectedBufferManager::GetProtectedNativePixmapFor`
- `arc::ProtectedBufferManager::GetProtectedNativePixmapFor`
  - `ImportDummyFd` calls
    `GbmSurfaceFactory::CreateNativePixmapForProtectedBufferHandle` with dummy
    params
    - `size` is `kDummyBufferSize` (16x16)
    - `format` is `gfx::BufferFormat::RGBA_8888`
    - `handle` has a dummy place with offset/stride/size 0
