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
  - `--no-sandbox`
  - `--enable-features=Vulkan`
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

- `CreateSharedImageFactory` or `LazyCreateSharedImageFactory` creates the gpu
  memory allocator
  - there can be multiple factories such as
  - `WrappedSkImageBackingFactory`
  - `OzoneImageBackingFactory`
- `SharedImageFactory::CreateSharedImage`
  - `GetFactoryByUsage` picks a factory based on buffer usage
  - `OzoneImageBackingFactory::CreateSharedImage` calls
    `GbmSurfaceFactory::CreateNativePixmap` on drm
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
