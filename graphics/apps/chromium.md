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
- build
  - `autoninja -C out/Default chrome`
- run
  - `out/Default/chrome`
- gn args
  - `build_with_chromium = true`
    - this is defined in `build/config/gclient_args.gni` generated from `DEPS`
  - defaults can be seen with `gn args out/Default --list`
    -  `is_official_build = false` and `is_debug = !is_official_build`
      - these control the optimization levels: debug, release, and official
    - `symbol_level = -1`, `v8_symbol_level = symbol_level`, and
      `blink_symbol_level = -1`
      - these control the debug symbols: auto (-1), none, min, full
    - `is_component_build = is_debug`
      - this controls shared or static libraries
      - set to true for faster linking and deploying
    - `dcheck_always_on = (build_with_chromium && !is_official_build)`
      - this controls whether `DLOG` and `DCHECK` are compiled in
    - `enable_nacl = true`
      - set to false for faster build
    - `use_goma = false`
      - set to true for faster build
    - `is_chrome_branded = false`
      - set to true for chrome branding
      - one big difference is in `ShouldUseFieldTrialTestingConfig`
        - on a chrome branded build, `fieldtrial_testing_config.json` is not
          applied by default
  - release build
    - `is_official_build = true`
  - recommended dev args
    - `is_debug = false`
    - `symbol_level = 1`
    - `v8_symbol_level = 0`
    - `blink_symbol_level = 0`
    - `is_component_build = true`
    - `enable_nacl = false` (cros requires nacl?)
    - `use_goma = true`
  - enable sw proprietary codecs
    - `proprietary_codecs = true'`
    - `ffmpeg_branding = "Chrome"'`

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
- if for whatever reasons emerge is used,
  - `cros_sdk --chrome-root ~/chromium --goma-dir ~/depot_tools/.cipd_bin`
    - this bind-mounts `~/chromium` to `$HOME/chrome_root` and
      `~/depot_tools/.cipd_bin` to `$HOME/goma` in the chroot
    - `goma_ctl goma_dir` reports the goma dir
  - `USE_GOMA=true CHROME_ORIGIN=LOCAL_SOURCE USE="-cros-debug -debug chrome_internal" FEATURES="noclean" emerge-$BOARD chromeos-chrome`
  - diffing `$HOME/chrome_root/src/build/args/chromeos/zork.gni` and
    `/var/cache/chromeos-chrome/chrome-src/src/out_zork/Release/args.gn`,
    these are the main differences
    - `dcheck_always_on = false`
    - `enable_hevc_parser_and_hw_decoder = false`
    - `internal_gles2_conform_tests = true`
    - `is_chrome_branded = true`
    - `is_cfi = true`
    - `is_official_build = true`
    - `use_thin_lto = true`

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

## Switches, Features, and Flags

- switches
  - switches are defined everywhere
    - `find -name '*_switches.cc'` for most of them
  - switches are parsed everwhere
    - `base::CommandLine::ForCurrentProcess()->HasSwitch(switches::kFooBar)`
    - `base::CommandLine::ForCurrentProcess()->GetSwitchValue(switches::kFooBar)`
- features
  - features are defined everywhere
    - `find -name '*_features.cc'` for most of them
    - they are defined by `BASE_FEATURE`
      - there are more than 4K features
      - each feature has a name and is enabled/disabled by default
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
  - <https://chromium.googlesource.com/chromium/src/+/main/testing/variations/>
    - `VariationsFieldTrialCreatorBase::SetUpFieldTrials` sets up field trials
    - on a on-chrome dev build, `FIELDTRIAL_TESTING_ENABLED` is set by default
      - `VariationsFieldTrialCreatorBase::ApplyFieldTrialTestingConfig`
        applies the testing config generated from
        `fieldtrial_testing_config.json`
- flags
  - flags are defined in `chrome/browser/about_flags.cc`
  - each flag is associated with a switch or a feature
    - it provides a means to set switches/features persistently
    - not all switches/features have associated flags
- `chrome://about`
  - `chrome://version`
  - `chrome://gpu`
  - `chrome://policy`
  - `chrome://flags`
- gpu
  - <https://chromium.googlesource.com/chromium/src/+/main/docs/gpu/debugging_gpu_related_code.md>
  - `--disable-gpu-sandbox`
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
    - `LOG(severity)` is enabled when `LOG_IS_ON(severity)` returns true
      - `--log-level=0` to enable `LOGGING_INFO`
  - `--v=1`
    - `VlogInfoFromCommandLine`
    - `VLOG(verbosity)` is enabled when `VLOG_IS_ON(verbosity)` returns true
      - `--v=1` to enable verbosity 1
  - `--disable-logging-redirect`
    - `RedirectChromeLogging`
    - on cros, this is the default (set by the session manager) on a test build
    - otherwise, logging is redirected to `/home/chronos/user/log/chrome`
      after user login
  - `DLOG` and `DVLOG` are enabled when `DCHECK_ALWAYS_ON` is enabled at
    compile time and the respective sevrity/verbosity is enabled at runtime

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

## GPU and GL

- GL is initialized in `GpuInit::InitializeAndStartSandbox`
- `gl::init::InitializeStaticGLBindingsOneOff` loads GL
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
- `gl::init::InitializeGLNoExtensionsOneOff` initializes GL
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
- I have a trace for chrome showing camera preview with effects
  - `cros_camera_service`: `Capture request` wakes up and calls
    `CameraDeviceAdapter::ProcessCaptureResult`
  - `cros_camera_service`: `DefaultCaptureR` wakes up and calls
    `StreamManipulatorManager::ProcessCaptureResultOnStreamManipulator`
  - `cros_camera_service`: `EffectsGlThread` wakes up and calls
    `EffectsStreamManipulatorImpl::ProcessCaptureResult` which calls
    `EffectsStreamManipulatorImpl::RenderEffect`
    - it calls `GpuImageProcessor::NV12ToRGBA` followed by `glFinish` to convert yuv to
      rgb
    - it calls `EffectsPipeline::ProcessFrame` which is another beast
      - it seems there are multiple `drishti_gl_runn` threads and most
        PROCESSes are on diffrent threads
        - `PROCESS::ImageHasMinWidthAndHeightCalculator`
        - `PROCESS::optionalimagedownscalecontainer`
        - `PROCESS::syrtisvulkansegmentationsubgraph`
        - `PROCESS::autolightpositionsubgraph`
        - `PROCESS::relightcontainer`
        - `PROCESS::bgreplacecontainer`
        - `PROCESS::bgblurcontainer`
        - `PROCESS::waitonrendercontainer`
        - `EffectsStreamManipulatorImpl::OnFrameProcessed`
      - it seems the effects pipeline has a queue.  `ProcessFrame` can submit
        framen N while the following `OnFrameProcessed` returns frame N-1 or
        even N-2.  For example,
        - 0.0ms: `ProcessFrame(N)`
        - 0.2ms-0.5ms: `PROCESS::ImageHasMinWidthAndHeightCalculator(N)` and
                       `PROCESS::optionalimagedownscalecontainer(N)`
        - 9.3ms: `PROCESS::waitonrendercontainer(N-1)` ends and
          `PROCESS::syrtisvulkansegmentationsubgraph(N)` begins
        - 9.5ms: `OnFrameProcessed(N-1)`
        - 23.0ms: `PROCESS::syrtisvulkansegmentationsubgraph(N)` ends
          - the gpu time is 9ms
        - 34.0ms-51.3ms: `PROCESS::relightcontainer(N)`
          - 1x `clEnqueueWriteBuffer`
          - Nx `clEnqueueNDRangeKernel` and 1x `clFlush`
            - 2x `vkQueueSubmit` on clvk
          - 1x `clEnqueueReadBuffer`
            - 1x `vkQueueSubmit` on clvk
          - the gpu times of the 3 queue submits are 4ms, 2ms, 5ms
        - 51.3ms-51.7ms: `PROCESS::bgreplacecontainer(N)` and `PROCESS::bgblurcontainer(N)`
        - 72.2ms: `PROCESS::waitonrendercontainer(N)` ends
    - on effects pipeline completion,
      `EffectsStreamManipulatorImpl::PostProcess` is called
      - this calls `GpuImageProcessor::RGBAToNV12` followed by `glFinish` to
        convert rgb back to yuv
    - on skyrim, it easily takes >30ms
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
- vaapi
  - it seems to be enabled by default on x11
    - `use_vaapi_x11`
    - `use_vaapi`
    - `VaapiVideoDecoder` feature is enabled by default and
      `VaapiVideoDecodeLinuxGL` feature is disabled by default
  - cmdline
    - `--ignore-gpu-blocklist`
- video decode trace view
  - renderer process
    - `MediaSource::StartAttachingToMediaElement`
    - `ChunkDemuxer::Initialize`
    - `DecoderSelector::SelectDecoder`
    - `VideoRendererImpl::Initialize`
    - `VideoDecoderStream::Read`
    - `VideoDecoderStream::ReadFromDemuxerStream`
    - `MojoVideoDecoder::Decode`
    - `MojoDecoderBufferWriter::Write`
    - `MojoDecoderBufferReader::Read`
    - `VideoDecoderStream::Decode`
  - gpu process
    - `MojoDecoderBufferReader::Read`
    - `MojoDecoderBufferWriter::Write`
    - `MojoVideoDecoderService::Decode`
    - `VideoDecoderPipeline::Decode`
    - `VideoDecoderPipeline::DecodeTask`
  - utility process
    - it seems to be doing OOP decoding where the real decoding happens on the
      utility process
    - `VideoDecoderPipeline::DecodeTask`
    - `VaapiVideoDecoder::HandleDecodeTask`
    - `VaapiVideoDecoder::Decode`
    - `VP9VaapiVideoDecoderDelegate::SubmitDecode`
    - `VaapiWrapper::MapAndCopyAndExecute`
  - summary
    - renderer `VideoDecoderStream::Read` returns a decoded frame
      - internally, it calls `VideoDecoderStream::Decode` to decode a frame
        which calls `MojoVideoDecoder::Decode`
      - `MojoDecoderBufferWriter::WriteDecoderBuffer` passes the encoded
        buffer over mojo
      - `ready_outputs_` holds the decoded frames
        - the type is `DecoderStream::Output` which is `VideoFrame`
        - it is updated by `DecoderStream<StreamType>::OnDecodeOutputReady`
    - gpu `MojoVideoDecoderService::Decode`
      - `MojoDecoderBufferReader::ReadDecoderBuffer` reads the buffer and
        invokes `VideoDecoderPipeline::Decode`
    - with oop decoding, gpu process offloads decoding to the utility process
- `VideoDecoderPipeline::Decode`
  - it calls `VideoDecoderPipeline::DecodeTask` on the runner thread which
    calls `decoder_->Decode`
  - `decoder_` is created by `VaapiVideoDecoder::Create`
  - when the frame is decoded/processed/converted, it calls
    - `VideoDecoderPipeline::OnFrameDecoded`
    - `VideoDecoderPipeline::OnFrameProcessed`
    - `VideoDecoderPipeline::OnFrameConverted`
      - this calls `client_output_cb_` to pass the decoded `VideoFrame` to the
        client
      - when the client is `MojoVideoDecoderService`,
        `MojoVideoDecoderService::OnDecoderOutput`
- `VaapiVideoDecoder::Decode`
  - it also has a `decoder_` that is created by `VaapiVideoDecoder::CreateAcceleratedVideoDecoder`
    - for vp9, it creates a `VP9VaapiVideoDecoderDelegate` and wraps the
      delegate in a `VP9Decoder`
  - it calls `VP9Decoder::Decode` which calls
    `VP9Decoder::DecodeAndOutputPicture` which calls
    `VP9VaapiVideoDecoderDelegate::CreateVP9Picture` and
    `VP9VaapiVideoDecoderDelegate::SubmitDecode`
  - `VP9VaapiVideoDecoderDelegate::CreateVP9Picture`
    - `vaapi_dec_->CreateSurface` is `VaapiVideoDecoder::CreateSurface`
      - it creates a `VideoFrame` using
        `PlatformVideoFramePool::GetFrame`
      - it imports the `VideoFrame` to a `VASurface` using
        `CreateNativePixmapDmaBuf` and
        `VaapiWrapper::CreateVASurfaceForPixmap`
  - `VP9VaapiVideoDecoderDelegate::SubmitDecode`
    - `VaapiWrapper::CreateVABuffer` create `VABuffer`s
    - `VaapiWrapper::MapAndCopyAndExecute`
      - uploads encoded data to `VABuffer`s
      - `vaBeginPicture`
      - `vaRenderPicture`
      - `vaEndPicture`
- `VideoFrame` and mojo
  - `media/mojo/mojom/BUILD.gn` says `media.mojom.VideoFrame` and
    `::scoped_refptr<::media::VideoFrame>` are convertible
  - the conversion uses `video_frame_mojom_traits.cc`'s `data` and `Read`
    methods
