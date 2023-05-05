Chromium Browser
================

## Build for Linux

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/linux/build_instructions.md>
- download
  - `mkdir ~/chromium && cd ~/chromium`
  - `fetch --nohooks chromium`
  - `cd src`
  - `./build/install-build-deps.sh`
  - `gclient runhooks`
- update
  - `git pull --rebase`
  - `gclient sync`
- setup
  - `gn gen out/Default`
    - the default will be a debug component build
      - see `build/config/BUILDCONFIG.gn`
      - `is_official_build = false`
      - `is_debug = !is_official_build`
      - `is_component_build = is_debug`
        - many shared libraries
      - `dcheck_always_on = (build_with_chromium && !is_official_build)`
        - `build_with_chromium = true` is from
          `build/config/gclient_args.gni`
  - faster build
    - disable nacl: `enable_nacl=false`
    - less debug symbols
      - `symbol_level=1`
      - `blink_symbol_level=0`
      - `v8_symbol_level=0`
  - release build
    - `is_debug = false`
    - `dcheck_always_on = false`
    - `is_official_build = true`
- build
  - `autoninja -C out/Default chrome`
- run
  - `out/Default/chrome`

## Build for ChromeOS

- <https://chromium.googlesource.com/chromiumos/docs/+/HEAD/simple_chrome_workflow.md>
- download
  - edit `.gclient`
    - `custom_vars`
      - `"cros_boards": "board1:board2"`
      - `"checkout_src_internal": True`
        - might require `gsutil.py config`
    - `target_os = ["chromeos"]`
  - `gclient sync`
- setup
  - `gn gen out_$BOARD/Release`
    - the default will be a release monolithic build
      - see `build/args/chromeos/$BOARD.gni`
      - `is_debug = false`
      - `is_component_build = is_debug`
      - `use_runtime_vlog = true`
  - interesting args
    - `use_goma = true` to enable goma
      - require goma setup
    - `is_component_build = true` to enable component build
      - faster link and deploy
    - `is_chrome_branded = true` to enable internal code?
      - is it more than branding?
    - debug component build
      - `is_debug = true`
      - `symbol_level = 1`
      - `blink_symbol_level = 0`
      - `v8_symbol_level = 0`
- build
  - `autoninja -C out_$BOARD/Release chrome nacl_helper`
- deploy
  - `./third_party/chromite/bin/deploy_chrome --build-dir=out_${BOARD}/Release --device=$DUT`
    - specify `--mount` to deploy to `/usr/local/opt/google/chrome` and bind
      mount to `/opt/google/chrome`
      - this is required when the rootfs is out of space
      - reboot will roll back to the original version (because the bind mount
        is gone)
    - specify `--deploy-test-binaries` to deploy
      `/usr/local/libexec/chrome-binary-tests`
    - `--nostrip` to avoid stripping
    - `--nostartui` to avoid restarting ui
- run
  - `ssh $DUT /usr/local/autotest/bin/autologin.py --url chrome://version`
- customize lauch flags and envvars
  - edit `/etc/chrome_dev.conf`
- logs
  - <https://chromium.googlesource.com/chromium/src/+/main/docs/chrome_os_logging.md>
  - `/var/log/ui/ui.LATEST`
    - early stdout/stderr from chrome and session manager
  - `/var/log/chrome/chrome`
    - chrome log after its logging subsystem has been initialized
- launch flags
  - <https://chromium.googlesource.com/chromiumos/platform2/+/refs/heads/main/login_manager/docs/flags.md>
  - `--enable-webgl-image-chromium`
    - webgl can render to scanout buffer
  - `--gpu-sandbox-failures-fatal=yes`
    - no thread allowed during `InitializeSandbox`
  - `--gpu-sandbox-start-early`
    - start sandboxing before gpu init, because gpu drivers may start
      threads
  - `--enable-logging`
  - `--log-level=1`
  - `--disable-logging-redirect`
  - `--login-manager`
    - chrome-as-a-login-manager
  - `--ml_service=enabled`
    - machine learning
  - `--enable-crashpad`
    - enable crashpad for crash reporting
  - `--enable-wayland-server`
    - enable exo wayland server
  - `--enable-native-gpu-memory-buffers`
  - `--video-capture-use-gpu-memory-buffer`

## `chrome://`

- `chrome://version`
- `chrome://gpu`
- `chrome://policy`
- `chrome://flags`
- gpu
  - <https://chromium.googlesource.com/chromium/src/+/main/docs/gpu/debugging_gpu_related_code.md>
  - `--no-sandbox`
  - `--enable-features=Vulkan`
  - `--disable-gpu-process-crash-limit`
- logging
  - `--enable-logging=stderr`
    - `DetermineLoggingDestination`
    - logging is enabled by default only on debug builds
  - `--log-level=0`
    - `InitChromeLogging`
    - `0` is `LOGGING_INFO` and is the lowest severity
  - `--v=1`
    - `VlogInfoFromCommandLine`
  - `--disable-logging-redirect`
    - `RedirectChromeLogging`
    - on cros, this is the default (set by the session manager) on a test build
    - otherwise, logging is redirected to `/home/chronos/user/log/chrome`
      after user login

## Directory Structure

- browsers
  - `//android_webview` provides Android's `WebView`
  - `//chrome` provides chrome browser on all platforms but ios
  - `//chromecast` provides Chrome Cast Receiver
  - `//content/shell` provides `content_shell`
    - this is a minimal browser mainly to develop `content`
  - `//fuchsia_web` provides Fuchsia's `WebEngine`
  - `//ios` provides chrome browser on ios
      - note that chrome on ios uses webkit rather than `//content`
  - `//weblayer` is `WebEngine`
    - this is a wrapper of `//content` and some browser extras
    - this is how apps embed chrome
- browser extras
  - `//apps` provides chrome app support
  - `//ash` provides cros's Aura Shell (system ui)
  - `//chromeos` provides cros-specific features
  - `//courgette` provides binary compress/delta for updates
  - `//extensions` provides chrome extension support
  - `//google_apis` provides google services
  - `//google_update` provides chrome updates
  - `//headless` provides headless support
  - `//native_client` provides NaCL support
  - `//native_client_sdk` provides NaCL SDK
  - `//pdf` provides pdf support
  - `//ppapi` provides ppapi support
  - `//printing` provides printer support
  - `//remoting` provides remote desktop support
  - `//rlz` provides some kind of fingerprinting?
  - `//sql` provides sql support
  - `//storage` provides storage support
- web platform
  - `//content` provides the web platform to build browsers
    - it is multi-process and sandboxed
    - it is unused on ios because ios requires all browsers to use webkit
- internal components
  - `//cc` provides chrome compositor
    - it organizes rectangular contents into a tree `LayerTreeHost` and
      converts the tree into a frame `CompositorFrame`
    - in the renderer process, it is used by `//third_party/blink`
    - in the browser process, it is used by `//ui`
  - `//components` provides utils useful for multiple subsystems
  - `//gin` provides a v8 binding
  - `//ipc` provides old ipc support
  - `//mojo` privodes new ipc support
  - `//net` provides network support
    - fetch a resource over http/ftp/etc.
  - `//services` provides foundation services
  - `//third_party/blink` provides the blink renderer
  - `//third_party/dawn` provides webgpu support
  - `//url` provides url parsing support
  - `//v8` provides v8 javascript engine
- low-level internal components
  - `//base` provides utils useful for the entire source tree
  - `//crypto` provides crypto support
  - `//dbus` provides dbus support
  - `//device` provides hw (bt, fido, gamepad, vr, etc.) support
  - `//gpu` provides gpu support
  - `//media` provides media (audio, video, camera, etc.) support
  - `//sandbox` provides sandboxing support
  - `//skia` provides skia with extensions
  - `//ui` provides ui support
    - `//ui/ozone` provides window system support
- misc
  - `//build` contains GN templates and configuration
  - `//build_overrides` contains GN rules to build chrome differently
  - `//buildtools` contains toolchain versions to download
  - `//codelabs` provides Chrome 101
  - `//docs` provides docs
  - `//infra` provides chrome infra
  - `//styleguide` provides the style guide
  - `//testing` provides testing tools
  - `//third_party` consists of third-party projects
  - `//tools` provides dev tools

## Multi-process architecture

- every process is an instance of `chrome`, the executable/dll
- the type (browser/render/gpu/...) of a process is determined by its command
  line switch, `-type <type>`

## Logging

- `base/logging.h`
  - `LOG(severity)` is enabled when `LOG_IS_ON(severity)` returns true
    - `--log-level=0` to enable `LOGGING_INFO`
  - `VLOG(verbosity)` is enabled when `VLOG_IS_ON(verbosity)` returns true
    - `--v=1` to enable verbosity 1
  - `DLOG` and `DVLOG` are enabled when `DCHECK_ALWAYS_ON` is enabled at
    compile time and the respective sevrity/verbosity is enabled at runtime

## Switches and Features

- switches
  - switches are defined everywhere
    - `find -name '*_switches.cc'` for most of them
  - switches are parsed everwhere
    - `base::CommandLine::ForCurrentProcess()->HasSwitch(switches::kFooBar)`
    - `base::CommandLine::ForCurrentProcess()->GetSwitchValue(switches::kFooBar)`
- features
  - features are defined everywhere
    - `find -name '*_features.cc'` for most of them
    - `--enable-features=Foo,Bar` is just another switch parsed by
      `FeatureList::InitializeInstance`
      - `FeatureList::RegisterOverride` is called on each enabled feature with
        `OVERRIDE_ENABLE_FEATURE` to update `overrides_`
  - `FeatureList::IsEnabled(features:kFoo)`
    - this calls `GetOverrideStateByFeatureName` to loop over `overrides_`
    - `OVERRIDE_ENABLE_FEATURE` overrides to enabled
    - `OVERRIDE_DISABLE_FEATURE` overrides to disabled
    - `OVERRIDE_USE_DEFAULT` uses the feature's `default_state`
      - `FEATURE_ENABLED_BY_DEFAULT` defaults to enabled
      - `FEATURE_DISABLED_BY_DEFAULT` defaults to disabled

## Startup

- `chrome/app` defines the entrypoint
  - `chrome_exe_main_aura.cc` defines `main` and calls `ChromeMain`
  - `chrome_main.cc` defines `ChromeMain` and calls `ContentMain` with a
    `ChromeMainDelegate`
- `content/app` defines the app process
  - `content_main.cc` defines `ContentMain` and calls `RunContentProcess`
  - `RunContentProcess` calls `ContentMainRunner::{Initialize,Run}`
  - `content_main_runner_impl.cc` defines `ContentMainRunnerImpl::{Initialize,Run}`
    - because `process_type.empty()`, it enters `RunBrowser`,
      `RunBrowserProcessMain`, and `BrowserMain`
- `content/browser` defines the browser process
  - `browser_main.cc` defines `BrowserMain` and calls
    `BrowserMainRunner::{Initialize,Run}`
  - `browser_main_runner_impl.cc` defines
    `BrowserMainRunnerImpl::{Initialize,Run}`
  - `GpuDataManagerImpl` defines how to launch the gpu process
    - `InitializeGpuModes` initializes the possible modes
      - `gpu::GpuMode::HARDWARE_GL` is always on
      - `gpu::GpuMode::HARDWARE_VULKAN` is based on
        `ParseVulkanImplementationName`, which checks
        `features::IsUsingVulkan` (`--enable-features=Vulkan`)
  - `GpuProcessHost::Get` launches the gpu process on demand
    - I guess this executes `chrome` again (or use the zygote) with
      `--type=gpu-process`.  `ContentMainRunnerImpl::Run` calls
      `RunOtherNamedProcessTypeMain` which calls `GpuMain`
- `content/gpu` defines the gpu process
  - `gpu_main.cc` defines `GpuMain` and calls `GpuInit::InitializeAndStartSandbox`
  - stack on cros
    - `base::RunLoop::Run()`
    - `content::GpuMain()`
    - `content::RunZygote()`
    - `content::RunOtherNamedProcessTypeMain()`
    - `content::ContentMainRunnerImpl::Run()`
    - `content::RunContentProcess()`
    - `content::ContentMain()`
    - `ChromeMain`
- `gpu/ipc` defines the `GpuInit` class
  - `service/gpu_init.cc` defines `GpuInit::InitializeAndStartSandbox`

## GPU and Initialization

- `GpuMain` is the entrypoint of the gpu process
  - `GpuPreferences` is parsed from `--gpu-preferences` passed in by the
    browser process
- `GpuInit::InitializeAndStartSandbox` initializes the gpu
  - if `--gpu-sandbox-start-early`, sandbox is started before gpu
    initializaton
    - `EnsureSandboxInitialized` calls `StartSandboxLinux` to start the
      sandbox
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
- `GpuChildThread` starts the gpu thread
  - `GpuChildThread` has a `VizMainImpl`
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

## GPU and Skia

- `SharedContextState::InitializeSkia` initializes skia
  - `InitializeGanesh` supports `GrContextType::kGL` and
    `GrContextType::kVulkan`
- if `GrContextType::kGL`, `GrDirectContext::MakeGL` is called
- if `GrContextType::kVulkan`,
  `VulkanInProcessContextProvider::InitializeGrContext` is called
  - which calls `GrDirectContext::MakeVulkan`

## GPU and WebGL

- WebKit expects the renderer to create a `WebKitClient`
- WebKit calls `WebKitClient::createGLES2Context` to create a `WebGLES2Context`
- the renderer implements `WebGLES2Context` with `ggl`, which in turns uses
  `gpu` for indirect rendering (renderer process to gpu process)
- the renderer sends commands to the gpu process (`chrome_gpu`)
  - the commands are handled by `GPUProcessor`
  - decoded GL commands are dispatched to `app/gfx/gl` directly
  - context management commands are sent to who?

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
- a `SharedImageBacking` is an external gpu resource (gl texture, vk image,
  etc.)
  - a factory is used for alloc/import
    - `CreateSharedImageFactory` creates the factory, `SharedImageFactory`
    - internally, there is an array of factories, `factories_`
      - `OzoneImageBackingFactory`
      - `WrappedSkImageBackingFactory`
      - `ExternalVkImageBackingFactory`
      - `EGLImageBackingFactory`
      - `GLTextureImageBackingFactory`
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
    - this calls `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)`
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

## Media

- <https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/media/README.md#playback>
  - `<video>` and `<audio>`
    - each `blink::HTMLMediaElement` owns a `blink::WebMediaPlayerImpl`, with
      call stack
      - `blink::WebMediaPlayerImpl`
      - `content::MediaFactory::CreateMediaPlayer`
      - `content::RenderFrameImpl::CreateMediaPlayer`
      - `blink::HTMLMediaElement`
  - `blink::WebMediaPlayerImpl`
    - owns a `media::PipelineController` to coordinate
      - `media::DataSource`
      - `media::Demuxer`
      - `media::Renderer`
    - calls `DemuxerManager::CreateDemuxer` to create a `media::Demuxer`
      - `media::FFmpegDemuxer` demuxes the video file into
        `media::DemuxerStream`s
    - `CreateRenderer` is called when the pipeline needs a renderer
      - calls `media::RendererFactorySelector::GetCurrentFactory` to get the
        current factory
      - usually reaches `media::RendererImplFactory::CreateRenderer` to
        create a `media::RendererImpl`
        - `AudioRendererImpl`,
        - `VideoRendererImpl`
        - `GpuMemoryBufferVideoFramePool`
  - `media::RendererImpl`
    - owns a `media::AudioRenderer` and a `media::VideoRenderer` to decode
      `media::DecoderStream`s
- from `--v=3` log,
  - `DecoderSelector<>::SelectDecoderInternal` tries all decoders one by one
  - once a decoder is selected, `DecoderStream<>::OnDecoderSelected` is called
  - when an encoded buffer is ready,
    `DecoderStream<StreamType>::OnBufferReady` calls
    `DecoderStream<StreamType>::Decode`
- ffmpeg
  - gn
    - `proprietary_codecs = true`
    - `ffmpeg_branding = "Chrome"`
- vaapi
  - it seems to be enabled by default on x11
    - `use_vaapi_x11`
    - `use_vaapi`
    - `VaapiVideoDecoder` feature is enabled by default and
      `VaapiVideoDecodeLinuxGL` feature is disabled by default
  - cmdline
    - `--ignore-gpu-blocklist`
