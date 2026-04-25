# Chromium GPU

## GPU and Initialization

- `GpuMain` is the entrypoint of the gpu process
  - `GpuPreferences` is parsed from `--gpu-preferences` passed in by the
    browser process
- `GpuInit::InitializeAndStartSandbox` initializes the gpu
  - `EnsureSandboxInitialized` calls `StartSandboxLinux` to start the
    sandbox
    - if `--gpu-sandbox-start-early`, sandbox is started before gpu
      initializaton
      - seccomp rules apply to the calling thread and its children
        - there is `SECCOMP_FILTER_FLAG_TSYNC` though
      - mesa creates threads during initialization
      - to make sure mesa threads are sandboxed, we need to start sandbox
        early
    - `SandboxLinux::Options` is initialized based on `gpu::GPUInfo` and
      `gpu::GpuPreferences`
    - `InitializeSandbox` calls `StartSeccompBPF` to start the sandbox
    - `GpuPreSandboxHook` is the pre-sandbox hook to configure the sandbox
      - `CommandSetForGPU` configures the allowed syscalls
      - `FilePermissionsForGpu` configures the allowed files
      - `LoadLibrariesForGpu` preloads libraries
        - I guess the seccomp rules prevent some driver library ctors from
          working and they are preloaded to mitigate
  - `OzonePlatform::InitializeForGPU` initializes ozone
    - `EnsureInstance` creates `OzonePlatform`
      - `PlatformObject<T>::Create` calls `GetOzonePlatformId` to determine
        the platform
      - on linux/wayland, `CreateOzonePlatformWayland`
        - `WAYLAND_GBM` is usually defined
      - on cros, `CreateOzonePlatformDrm`
    - `OzonePlatformWayland::InitializeGPU`
      - `GetDrmRenderNodePath` returns the render node path
      - `WaylandBufferManagerGpu` is created
      - `WaylandSurfaceFactory` is created
      - `WaylandOverlayManager` is created
    - `OzonePlatformDrm::InitializeGPU`
      - `DrmThreadProxy` is created
      - `GbmSurfaceFactory` is created
      - `DrmOverlayManagerGpu` is created
  - `gl::init::InitializeStaticGLBindingsOneOff` loads EGL/GLES
    - nowadays, `kGLImplementationEGLANGLE` is used and this loads angle
      shipped with chrome
    - `GetRequestedGLImplementation` returns the `GLImplementationParts`
      - `GetAllowedGLImplementations` returns allowed implementations and is
        platform-specific on ozone
      - because angle and cmd passthrough are both enabled, angle
        implementations are moved to the head of the vector
      - if `--use-gl=foo` is specified,
        `GetRequestedGLImplementationFromCommandLine` return `foo` impl
      - otherwise, it falls back to the first implementation
        - on drm, it is `kGLImplementationEGLANGLE` and
          `ANGLEImplementation::kDefault`
        - on wayland, it is `kGLImplementationEGLANGLE` and
          `gl::ANGLEImplementation::kOpenGL`
    - `InitializeStaticGLBindingsImplementation`
      - this calls `GLOzoneEGL::InitializeStaticGLBindings` on drm and wayland
      - on both platforms, `LoadDefaultEGLGLES2Bindings` is used to dlopen
        angle's `libEGL.so` and `libGLESv2.so`
  - `gl::init::InitializeGLNoExtensionsOneOff` initializes EGL display
    - on ozone, `InitializeGLOneOffPlatform` calls
      - `GetDisplayInitializationParams` to collect supported `DisplayType`s
        - it often returns `ANGLE_OPENGL`, `ANGLE_OPENGLES`, and `ANGLE_VULKAN`
        - it checks for allowed implementations and angle's EGL client
          extensions
          - `EGL_ANGLE_platform_angle`
          - `EGL_ANGLE_platform_angle_opengl`
          - `EGL_ANGLE_platform_angle_vulkan`
          - `EGL_ANGLE_platform_angle_device_type_egl_angle`
      - `GLOzoneEGL::InitializeGLOneOffPlatform` to initialize a display
        - it calls `GLDisplayEGL::Initialize` which calls `eglInitialize`
  - `use_passthrough_cmd_decoder` means chrome decodes and passes through
    gles2 cmds to the driver without validation
    - the idea is, when the driver is angle, angle will validate
  - `CollectGraphicsInfo` creates a temp EGL surface/context to query driver
    GL caps
  - `ComputeGpuFeatureInfo` is based on driver GL caps with driver WAs applied
  - if vulkan is enabled, `InitializeVulkan` loads vk and initializes a vk
    instance
    - `VulkanInstance::InitializeInstance` reuses the vk instance from
      angle when `VulkanFromANGLE` is enabled
  - `gl::init::InitializeExtensionSettingsOneOffPlatform` parses EGL
    extensions
  - `gl::init::CreateOffscreenGLSurface` creates a offscreen EGL surface
    - it is either pbuffer or no surface at all if the driver supports
      surfaceless
- `GpuChildThread` starts the gpu (main) thread
  - `GpuChildThread` has a `VizMainImpl`
  - `VizMainImpl::VizMainImpl` creates a `GpuServiceImpl`

## GPU and Mojo

- viz privileged interfaces
  - these are between browser and gpu processes
  - `interface GpuHost`
    - browser process has a `content::GpuProcessHost` to manage the gpu
      process
    - `content::GpuProcessHost` has a `viz::GpuHostImpl` to implement
      `mojom::GpuHost` interface
  - `interface VizMain`
    - gpu process has a `viz::VizMainImpl` to implement `mojom::VizMain`
      interface
  - `interface GpuService`
    - gpu process also a `viz::GpuServiceImpl` to implement
      `mojom::GpuService` interface
    - browser `GpuHostImpl::GpuHostImpl` calls gpu
      `VizMainImpl::CreateGpuService` to initialize `viz::GpuServiceImpl`
- public interfaces
  - these are between browser and renderer processes
  - `interface Gpu` and `interface GpuMemoryBufferFactory`
    - browser process has a `RenderProcessHostImpl` to manage each renderer
      process
    - each `RenderProcessHostImpl` has a `viz::GpuClient` to implement both
      `mojom::Gpu` and `mojom::GpuMemoryBufferFactory` interfaces
- gpu channel establishment
  - renderer `RenderThreadImpl::Init` calls `viz::Gpu::Create` to create a
    `viz::Gpu`
  - renderer `content::RenderThreadImpl::EstablishGpuChannel`
  - renderer `viz::Gpu::EstablishGpuChannel`
  - renderer `viz::Gpu::GpuPtrIO::EstablishGpuChannel`
  - browser `viz::GpuClient::EstablishGpuChannel`
  - browser `viz::GpuHostImpl::EstablishGpuChannel`
  - gpu `viz::GpuServiceImpl::EstablishGpuChannel`
  - gpu `gpu::GpuChannelManager::EstablishChannel`
    - create a `gpu::GpuChannel` in the gpu process
    - create a `GpuChannelHost` in the renderer process
- gpu memory buffer
  - renderer `viz::Gpu::Gpu` also creates a `ClientGpuMemoryBufferManager`
    - it wraps `mojom::GpuMemoryBufferFactory` and `mojom::ClientGmbInterface`
    - both interfaces provide the same functionality, except
      `mojom::GpuMemoryBufferFactory` communicates with the browser process
      and `mojom::ClientGmbInterface` communicates with the gpu process
    - nowadays, `UseClientGmbInterface` is on by default
- gpu command buffer
  - renderer creates a `ContextProviderCommandBuffer`
    - it provides a gles context or a raster context over a command buffer to
      the gpu process
    - it internally has a `CommandBufferProxyImpl`
  - renderer `gpu::CommandBufferProxyImpl::Initialize` calls `CreateCommandBuffer`
  - gpu `gpu::GpuChannel::CreateCommandBuffer` creates a `CommandBufferStub`
    to handle mojo messages
    - it implements `mojom::CommandBuffer` interface
    - it is one of `WebGPUCommandBufferStub`, `RasterCommandBufferStub`, or
      `GLES2CommandBufferStub`

## GPU and Skia

- `SharedContextState::InitializeSkia` initializes skia
  - `InitializeGanesh` supports `GrContextType::kGL` and
    `GrContextType::kVulkan`
- if `GrContextType::kGL`, `GrDirectContext::MakeGL` is called
- if `GrContextType::kVulkan`,
  `VulkanInProcessContextProvider::InitializeGrContext` is called
  - which calls `GrDirectContext::MakeVulkan`
- renderer compositor and remote raster (OOP-R)
  - blink uses "paint ops" rather than `SkPicture` to serialize commands
  - `TileManager::CreateRasterTask` creates raster tasks
    - it looks like a `RasterTaskImpl` is created for a `RasterSource` and a
      `RasterBuffer`
  - `RasterTaskImpl::RunOnWorkerThread` calls `RasterBuffer::Playback`
  - `GpuRasterBufferProvider::RasterBufferImpl::Playback` calls
    `BeginRasterCHROMIUM`, `RasterCHROMIUM`, and `EndRasterCHROMIUM` of
    `RasterInterface`
  - `RasterImplementation` implements `RasterInterface` over command buffers
  - the service uses `RasterDecoderImpl`
    - `RasterDecoderImpl::DoRasterCHROMIUM` deserializes `cc::PaintOp` and
      calls their `Raster` method to draw with skia
- display compositor and `SkiaRenderer`
  - the browser process calls `CreateRootCompositorFrameSink` to create a
    `RootCompositorFrameSinkImpl` in the gpu process
  - `RootCompositorFrameSinkImpl::Create` `std::make_unique<Display>` a
    `viz::Display` which `std::make_unique<SkiaRenderer>` a `SkiaRenderer`
  - when it's time to swap, I guess `DisplayScheduler::DrawAndSwap` is called
    - `DisplayScheduler::DrawAndSwap` calls `Display::DrawAndSwap`
    - `Display::DrawAndSwap` calls `DirectRenderer::DrawFrame`
    - `SkiaRenderer` draws with skia
      - `DirectRenderer::DrawFrame` invokes these methods from `SkiaRenderer`
        - `BeginDrawingFrame`
        - `BeginDrawingRenderPass`
        - `DoDrawQuad`
        - `FinishDrawingRenderPass`
        - `FinishDrawingFrame`
        - more
      - the `SkCanvas` is from `SkiaOutputSurfaceImpl`
        - `SkiaOutputSurfaceImpl::BeginPaintCurrentFrame` returns the canvas
          - the canvas is owned by `GrDeferredDisplayListRecorder`
          - all ops are recorded and deferred
        - `SkiaOutputSurfaceImpl::EndPaint` calls
          `SkiaOutputSurfaceImplOnGpu::FinishPaintCurrentFrame` on the gpu
          thread
        - ultimately, the gpu thread calls `SkiaOutputDevice::Draw`
- `RasterDecoderImpl::DoBeginRasterCHROMIUM` begins a raster pass
- `RasterDecoderImpl::DoRasterCHROMIUM` replays the raster ops
- `RasterDecoderImpl::DoEndRasterCHROMIUM` ends a raster pass
  - `FlushWriteAccess` calls `skgpu::ganesh::Flush` to flush the surface
    - `skgpu::ganesh::Flush` calls `GrDirectContext::flush`
    - `GrDirectContextPriv::flushSurfaces` calls
      `GrDrawingManager::flushSurfaces` and `GrDrawingManager::flush`
    - `GrDrawingManager::executeRenderTasks` calls `GrOp::execute`
      - `OpsTask::onExecute`
        - `create_render_pass` creates a render pass
          - `GrVkGpu::onGetOpsRenderPass` calls `GrVkOpsRenderPass::set`
        - `GrOpsRenderPass::begin` begins a render pass
        - various `GrOp::execute` to draw
        - `GrOpsRenderPass::end` calls `GrVkOpsRenderPass::onEnd`
          - `GrVkSecondaryCommandBuffer::end` ends the secondary cmdbuf
        - `GrVkGpu::submit` calls
          - `GrVkOpsRenderPass::submit` execs secondary cmdbuf from primary
            cmdbuf and ends the vk render pass
          - `GrVkOpsRenderPass::reset` resets the states
  - `SharedContextState::SubmitIfNecessary` may or may not call
    `GrDirectContext::submit`
- `GrCacheController::PerformDeferredCleanup` is called periodically
  - `GrDirectContext::performDeferredCleanup`
    - `GrDirectContext::checkAsyncWorkCompletion`
    - `GrVkGpu::checkFinishedCallbacks`
    - `GrVkResourceProvider::checkCommandBuffers` checks if any cmdbuf has
      completed and releases the resources
  - chrome may call `GrDirectContext::checkAsyncWorkCompletion` directly too

## GPU and IPC

- An `IPC::Channel` is created with a channel ID and consists of `socketpair`
  - A channel ID is a string of this form `"%d.%p.%d" % (pid, this, random())`

## GPU and Memory

- note that only the gpu process has access to the gpu
  - other processes can still receive/send gpu bos
  - on linux, they also have cpu access in the form of `ClientNativePixmap`
- a `gfx::NativePixmap` is a gpu bo in the gpu process on linux
  - alloc
    - `GbmDevice::CreateBuffer` allocates a `GbmBuffer` (a `gbm_bo`)
    - `GbmBuffer::ExportHandle` exports a `gfx::NativePixmapHandle`
    - a `gfx::NativePixmap` can be created from the handle by
      `NativePixmapDmaBuf()`
  - import
    - when the gpu process gets an external memory, such as
      `VADRMPRIMESurfaceDescriptor` returned from `vaExportSurfaceHandle`, it
      wraps the external memory in a `gfx::NativePixmapHandle`
  - export
    - `NativePixmap::ExportHandle` exports a `gfx::NativePixmapHandle`
- a `gfx::GpuMemoryBuffer` is a platform-agnostic gpu bo
  - no direct alloc
  - import
    - a `gfx::GpuMemoryBufferHandle` of type `gfx::NATIVE_PIXMAP` can wrap a
      `gfx::NativePixmapHandle`
    - a `gfx::GpuMemoryBuffer` can be created from the handle by
      `GpuMemoryBufferSupport::CreateGpuMemoryBufferImplFromHandle`
  - export
    - `GpuMemoryBuffer::CloneHandle` exports a `gfx::GpuMemoryBufferHandle`
  - `enum GpuMemoryBufferType`
    - `EMPTY_BUFFER` has no backing
    - `SHARED_MEMORY_BUFFER` is generic and the handle is a
      `UnsafeSharedMemoryRegion`
      - it is a cpu shared memory
      - on linux, it is a shm under `/dev/shm`
    - `IO_SURFACE_BUFFER` is for mac and the handle is a `ScopedIOSurface`
    - `NATIVE_PIXMAP` is for linux and the handle is a `NativePixmapHandle`
    - `DXGI_SHARED_HANDLE` is for windows and the handle is a `ScopedHandle`
      and a `DXGIHandleToken`
    - `ANDROID_HARDWARE_BUFFER` is for android and the handle is a
      `ScopedHardwareBufferHandle`
- a `SharedImageBacking` is an external gpu resource (gl texture, vk image,
  etc.)
  - a factory is used for alloc/import
    - `CreateSharedImageFactory` creates the factory, `SharedImageFactory`
    - internally, there is an array of factories, `factories_`
      - `SharedMemoryImageBackingFactory`
        - `SharedMemoryImageBackingFactory::IsSupported` supports
          `SHARED_MEMORY_BUFFER` and `SHARED_IMAGE_USAGE_CPU_WRITE`
      - `WrappedSkImageBackingFactory`
        - `WrappedSkImageBackingFactory::IsSupported` supports `EMPTY_BUFFER`
          (i.e., it allocs a skia image instead of import)
      - `GLTextureImageBackingFactory`
        - `GLTextureImageBackingFactory::IsSupported` supports `EMPTY_BUFFER`
          (i.e., it allocs a gl texture instead of import)
      - `AngleVulkanImageBackingFactory` if
        `--enable-features=Vulkan,VulkanFromANGLE`
        - `AngleVulkanImageBackingFactory::IsSupported` supports
          `EMPTY_BUFFER` and `NATIVE_PIXMAP` (i.e., it supports alloc and
          import)
      - `EGLImageBackingFactory`
        - `EGLImageBackingFactory::IsSupported` supports `EMPTY_BUFFER` (i.e.,
          it supports alloc)
      - `OzoneImageBackingFactory`
        - `OzoneImageBackingFactory::IsSupported` supports `EMPTY_BUFFER` and
          `NATIVE_PIXMAP` (i.e., it supports alloc and import)
      - `ExternalVkImageBackingFactory`
        - `ExternalVkImageBackingFactory::IsSupported` supports `EMPTY_BUFFER`
          and whatever the vulkan impl supports (e.g., `NATIVE_PIXMAP` if the
          vk driver supports dma-buf export/import)
  - `SharedImageFactory::CreateSharedImage` internally allocs/imports a
    `SharedImageBacking`
    - `GetFactoryByUsage` picks a factory based on buffer usage
    - `OzoneImageBackingFactory::CreateSharedImage` calls
      `GbmSurfaceFactory::CreateNativePixmap` or
      `GbmSurfaceFactory::CreateNativePixmapFromHandle` on drm
      - `DrmThreadProxy::CreateBuffer`
      - `DrmThread::CreateBuffer`
        - `window->GetController()` may be null and there will be no modifier
        - otherwise, on zork, `modetest` says `AR24` supports
          - `AMD_GFX9,GFX9_64K_S_X,DCC,DCC_INDEPENDENT_64B,DCC_MAX_COMPRESSED_BLOCK=64B,PIPE_XOR_BITS=2,BANK_XOR_BITS=0,RB=0`
          - `AMD_GFX9,GFX9_64K_S_X,DCC,DCC_RETILE,DCC_INDEPENDENT_64B,DCC_MAX_COMPRESSED_BLOCK=64B,PIPE_XOR_BITS=2,BANK_XOR_BITS=0,RB=1,PIPE_2`
          - `AMD_GFX9,GFX9_64K_S_X,PIPE_XOR_BITS=2,BANK_XOR_BITS=0`
          - `AMD_GFX9,GFX9_64K_S`
          - `LINEAR`
        - I've seen `0x200000440417901` which is the second modifier above
          - because of `DCC` and `DCC_RETILE`, there are 3 planes
          - plane 0 is the main surface
          - plane 1 is the displayable dcc surface
          - plane 2 is the pipe-aligned dcc surface
        - also `0x200000000401901` which is the third modifier above
      - `CreateBufferWithGbmFlags`
      - `GbmDevice::CreateBufferWithModifiers`
    - `WrappedSkImageBackingFactory::CreateSharedImage` calls
      `WrappedSkImageBacking::Initialize`
      - this calls skia's `createBackendTexture`
      - there is no import support in this factory
- when skia-vk imports a shared image for composition,
  - there is this stack
    - `gpu::VulkanImage::InitializeFromGpuMemoryBufferHandle()`
    - `gpu::VulkanImage::CreateFromGpuMemoryBufferHandle()`
    - `ui::VulkanImplementationGbm::CreateImageFromGpuMemoryHandle()`
    - `gpu::OzoneImageBacking::ProduceSkiaGanesh()`
    - `gpu::SharedImageBacking::ProduceSkia()`
    - `gpu::SharedImageManager::ProduceSkia()`
    - `gpu::SharedImageRepresentationFactory::ProduceSkia()`
    - `viz::SkiaOutputSurfaceImplOnGpu::FinishPaintRenderPass()`
  - skia-gl instead uses `GLOzoneEGLGbm::ImportNativePixmap`
    - `NativePixmapEGLBinding::InitializeFromNativePixmap` calls
      `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)`
    - when the `NativePixmap` does not have a modifier
      (`NativePixmapHandle::kNoModifier` is `DRM_FORMAT_MOD_INVALID`), there
      is no `EGL_DMA_BUF_PLANE*_MODIFIER_*_EXT`
- Formats
  - ui format `BGRA_8888` or `RGBA_8888`, modifiers are
    - `0x200000000401901`, 1 plane (amd without dcc)
    - `0x200000440417901`, 3 planes (amd with dcc)
    - `0x100000000000001`, 1 plane (`I915_FORMAT_MOD_X_TILED`)
    - `0x100000000000006`, 2 planes (`I915_FORMAT_MOD_Y_TILED_GEN12_RC_CCS`)
      - before gen12, it was `0x100000000000002` (`I915_FORMAT_MOD_Y_TILED`)
  - video format `YUV_420_BIPLANAR`, modifiers are
    - `0`, 2 planes (linear)
    - `0x100000000000002`, 2 planes (`I915_FORMAT_MOD_Y_TILED`)
  - camera format `YUV_420_BIPLANAR`, modifiers are
    - `0`, 2 planes (linear)
- on wayland,
  - when chrome draws and presents,
    - `viz::SkiaOutputDeviceBufferQueue::RecreateImages` creates a few
      `SharedImageBacking`
      - it allocates a few `NativePixmap` from gbm
    - skia draws to the shared image
    - `viz::SkiaOutputDeviceBufferQueue::ScheduleOverlays` presents
      - it reaches `GbmSurfacelessWayland::ScheduleOverlayPlane` and then
        `GbmPixmapWayland::ScheduleOverlayPlane`
      - `GbmPixmapWayland::CreateDmabufBasedWlBuffer` exports dmabuf for
        `wl_buffer`
  - when sw decode a video stream,
    - when an encoded buffer is ready,
      `media::DecoderStream<>::OnBufferReady` calls down to
      `media::FFmpegVideoDecoder::Decode` to decode the buffer
    - after decoding, `media::FFmpegVideoDecoder::OnNewFrame` calls
      `media::DecoderStream<>::OnDecodeOutputReady` to add the decoded buffer
      back to the decoder stream
    - because a gmb pool is enabled,
      `GpuMemoryBufferVideoFramePool::MaybeCreateHardwareFrame` allocates a
      `GpuMemoryBuffer` and copies the decoded buffer to the gmb
      - the allocation goes through
        `GpuVideoAcceleratorFactoriesImpl::CreateGpuMemoryBuffer`,
        `ClientGpuMemoryBufferManager::CreateGpuMemoryBuffer`
      - the client side uses `viz::Gpu` and the service side uses
        `viz::GpuServiceImpl`
      - `gpu::GpuMemoryBufferFactoryNativePixmap` allocates through ozone
    - after copying to gmb,
      `GpuMemoryBufferVideoFramePool::PoolImpl::BindAndCreateMailboxesHardwareFrameResources` creates a shared image
      - the client side uses `gpu::ClientSharedImageInterface`
      - the service side uses `gpu::SharedImageStub` and `gpu::SharedImageFactory`
        - this is an import operation

## GPU and Fences

- a `gfx::GpuFenceHandle` is a sync fd on linux
  - on x11, it is unsued
  - on wayland, it is a sync fd
    - present in-fence
      - `GbmSurfacelessWayland::ScheduleOverlayPlane` gets the in-fence from viz
      - `GbmPixmapWayland::ScheduleOverlayPlane` forwards the in-fence to
        `GbmSurfacelessWayland::QueueWaylandOverlayConfig` in
        `WaylandOverlayConfig`
      - the acquire fence is passed to the compositor using
        `zwp_linux_explicit_synchronization_v1`
    - present out-fence
      - `WaylandSurface::FencedRelease` handles the `fenced_release` event and
        calls `WaylandFrameManager::OnExplicitBufferRelease`
      - the out-fence is merged to `WaylandFrame::merged_release_fence_fd`
      - `GbmSurfacelessWayland::OnSubmission` passes the out-fence to viz in
        `SwapCompletionResult`
  - on drm, it is also a sync fd
- a `gpu::SemaphoreHandle` is an fd on linux
  - it is used with winsys and must be sync fds
  - it is also used in `gpu::ExternalSemaphore` for vk-gl interop and must be
    opaque fds
  - boom!
- a `gfx::GpuFence` wraps a `gfx:GpuFenceHandle` and supports a few sync fd
  operations on linux
  - `GpuFence::Wait` calls `sync_wait`
  - `GpuFence::GetStatusChangeTime` calls `sync_fence_info` and `sync_pt_info`
- examples
  - `HardwareDisplayController::SchedulePageFlip` does an atomic commit
    - `HardwareDisplayPlaneManagerAtomic::Commit` returns the out fence
    - when the submit callback is `GbmSurfaceless::OnSubmission`, the out
      fence is returned to viz in `gfx::SwapCompletionResult`
  - `gl::GLFence::CreateForGpuFence` is called after submitting GL work
    - well, only if `gl::GLFence::IsGpuFenceSupported` returns true
    - on android, it uses `EGL_SYNC_NATIVE_FENCE_ANDROID` and
      `eglDupNativeFenceFDANDROID`
    - on linux, it relies on implicit fencing
  - in exo, `linux_surface_synchronization_set_acquire_fence` gets the
    in-fence from clients
    - `Surface::SetAcquireFence` updates `pending_state_.acquire_fence`
    - `Buffer::Texture::UpdateSharedImage` sends an ipc which is handled by
      `SharedImageStub::OnUpdateSharedImage`
    - on ozone, `OzoneImageBacking::Update` saves the in-fence to
      `external_write_fence_`
- `OzoneImageBacking` maintains fences for the shared image
  - `write_fence_` is the fence of the writer
  - `external_write_fence_` is the fence of an external writer (from exo, that
    is, wayland clients)
  - `read_fences_` are fences of the readers
  - `BeginAccess` returns the fences to the caller
  - `EndAccess` adds fences to the shared image
- `SkiaOutputSurfaceImplOnGpu::SwapBuffers`
  - `output_device_` is `SkiaOutputDeviceBufferQueue`
  - this calls `SkiaOutputDevice::Submit` which calls
    `GrDirectContext::submit` to submit the work to gpu
  - in `SkiaOutputSurfaceImplOnGpu::PostSubmit`,
    - `SkiaOutputDeviceBufferQueue::ScheduleOverlays` is called to schedule
      overlays
      - `gl::GLFence::CreateForGpuFence` is called to get the gpu fence
    - `SkiaOutputDeviceBufferQueue::SchedulePrimaryPlane` is called to
      schedule the primary plane
      - `PresenterImageGL::BeginPresent` gets an acquire fence
      - `OutputPresenterGL::SchedulePrimaryPlane` passes down the acquire
        fence
  - in `SkiaOutputDeviceBufferQueue::DoFinishSwapBuffers`,
    `SwapCompletionResult::release_fence` is the release fence
    - `PresenterImageGL::EndPresent` is called with the release fence
- on vulkan,
  - `SkiaOutputSurfaceImplOnGpu::FinishPaintCurrentFrame` calls
    `viz::SkiaOutputDevice::BeginScopedPaint`
    - `viz::OutputPresenter::Image::BeginWriteSkia` gets the begin and end
      semaphores; adds the begin semaphores to `SkSurface`; saves the end
      semaphores created by
      - `ui::VulkanImplementationWayland::CreateExternalSemaphore()`
      - `gpu::SkiaVkOzoneImageRepresentation::BeginAccess()`
      - `gpu::SkiaVkOzoneImageRepresentation::BeginWriteAccess()`
      - `gpu::SkiaGaneshImageRepresentation::BeginScopedWriteAccess()`
      - `gpu::SkiaGaneshImageRepresentation::BeginScopedWriteAccess()`
      - `viz::OutputPresenter::Image::BeginWriteSkia()`
    - `viz::OutputPresenter::Image::EndWriteSkia` submits the end semaphores
      to skia; `scoped_skia_write_access_.reset()` exports the end semaphores
      and add them to the external image
      - `ui::VulkanImplementationWayland::GetSemaphoreHandle()`
      - `gpu::SkiaVkOzoneImageRepresentation::EndAccess()`
      - `gpu::SkiaVkOzoneImageRepresentation::EndWriteAccess()`
      - `gpu::SkiaImageRepresentation::ScopedWriteAccess::~ScopedWriteAccess()`
      - `gpu::SkiaGaneshImageRepresentation::ScopedGaneshWriteAccess::~ScopedGaneshWriteAccess()`
      - `gpu::SkiaGaneshImageRepresentation::ScopedGaneshWriteAccess::~ScopedGaneshWriteAccess()`

## GPU and Vulkan

- `gpu/ipc`
  - `GpuInit::InitializeAndStartSandbox`
    - it calls `ComputeGpuFeatureInfo` to parse the command line and the
      preference into `GpuFeatureInfo`
      - `--enable-features=Vulkan` forces vulkan
    - if `GPU_FEATURE_TYPE_VULKAN`, `GpuInit::InitializeVulkan` initializes vk
      - `gpu::CreateVulkanImplementation` calls
        `GbmSurfaceFactory::CreateVulkanImplementation` if using ozone drm
- `chrome/browser`
  - `kFeatureEntries`
  - `kEnableVulkanName`
- `ui/ozone`
- `ui/gl`
  - angle-on-vulkan
- `services/viz`
- `media`
- `gpu/command_buffer`
- `gpu/config`
- `gpu/vulkan`
- `content/browser`
- `content/gpu`
- `content`
- `components/exo`
- `components/viz`
