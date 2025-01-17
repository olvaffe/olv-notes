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
  - `gclient sync -D`
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
        - official build is very slow to build
        - debug build is very slow to run
        - use release build!
    - `symbol_level = -1`, `v8_symbol_level = symbol_level`, and
      `blink_symbol_level = -1`
      - these control the debug symbols: auto (-1), none (0), min (1),
        full (2)
    - `is_component_build = is_debug`
      - this controls shared or static libraries
      - set to true for faster linking and deploying
      - but it appears to introduce races in `VaapiWrapper`
    - `dcheck_always_on = (build_with_chromium && !is_official_build)`
      - this controls whether `DLOG` and `DCHECK` are compiled in
    - `enable_nacl = false`
      - only for chromeos
    - `use_remoteexec = false`
      - set to true for faster build
      - was `use_goma`
    - `is_chrome_branded = false`
      - set to true for chrome branding
      - one big difference is in `ShouldUseFieldTrialTestingConfig`
        - on a chrome branded build, `fieldtrial_testing_config.json` is not
          applied by default
  - recommended dev args
    - `is_chrome_branded = true`
    - `is_debug = false`
    - `use_remoteexec = true`
    - `is_component_build = true`
      - seems to have issues with `VaapiWrapper` though
    - `dcheck_always_on = true`
    - `symbol_level = 1`
    - `v8_symbol_level = 0`
    - `blink_symbol_level = 0`
  - for a chromium build where `is_chrome_branded = false`
    - `disable_fieldtrial_testing_config = true`
    - `proprietary_codecs = true'`
    - `ffmpeg_branding = "Chrome"'`
    - see also
      <https://gitlab.archlinux.org/archlinux/packaging/packages/chromium/-/blob/main/PKGBUILD>
- packaging
  - `ninja -C out/Default installer` generates `out/Default/*.deb`
  - arch package has its own rules for packaging
  - cros uses
    <https://chromium.googlesource.com/chromiumos/chromite.git/+/refs/heads/main/lib/chrome_util.py>
    - `StageChromeFromBuildDir` copies files from the build dir to the staging
      dir
    - `_COPY_PATHS_CHROME` is the list of files to copy

## Build for ChromeOS

- <https://chromium.googlesource.com/chromiumos/docs/+/HEAD/simple_chrome_workflow.md>
- download
  - edit `.gclient`
    - `custom_vars`
      - `"cros_boards": "board1:board2"`
      - `"checkout_src_internal": True`
        - might require `gcloud auth login`
      - `"download_remoteexec_cfg": True`
    - `target_os = ["chromeos"]`
    - this affects what `DEPS` does when `gclient sync` runs
      - e.g., it runs `cros chrome-sdk` to
        - download the toolchain to `src/build/cros_cache`
        - download gn args to `src/build/args/chromeos`
        - create `src/out_$BOARD/Release/args.gn`
  - `gclient sync -D`
- setup
  - `gn gen out_$BOARD/Release`
    - the pre-generated `args.gn` has
      `import("//build/args/chromeos/$BOARD.gni")`
    - these additional changes are recommended
      - `is_chrome_branded = true`
      - `use_remoteexec = true` for faster build
      - `dcheck_always_on = false` if don't need dchecks
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
  - `//skia` provides skia extensions
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
    - `//third_party/angle` provides gles (for webgl and skia-gl)
    - `//third_party/blink` provides the blink renderer
    - `//third_party/dawn` provides webgpu support
    - `//third_party/skia` provides 2D graphics (with gles and vk backends)
  - `//tools` provides dev tools

## Versioning

- `chrome/VERSION`
  - `MAJOR` is incremented every 4 weeks
  - `MINOR` is always 0
  - `BUILD` is incremented every 12 hours
  - `PATCH` is not used on main
- branching
  - when `BUILD` is incremented to `X+1`, it concludes the development of
    BUILD `X`
  - `refs/branch-heads/<X>` branch is created from some earlier commit
    - the branch is not fetched by default
  - a commit to add `chrome_branch_deps.json` is added and tagged on the
    branch
  - `PATCH` version is incremented once a day (or on-demand) if there are
    changes

## Multi-process architecture

- every process is an instance of `chrome`, the executable/dll
- the type (browser/render/gpu/...) of a process is determined by its command
  line switch, `--type <type>`
- initially,
  - the gpu process provides remote gles
    - clients use a gles implementation that serializes commands into
      command buffers
    - the gpu process deserializes command buffers and executes the gles
      command
    - that is, a client serializes commands into `CommandBufferProxyImpl`
      and the gpu process deserializes them from `GLES2CommandBufferStub`
  - the per-tab renderer process runs the renderer compositor
    - the renderer comopsitor composites html elements into a frame
    - it uses skia-over-gles which the gles impl is remote
    - frames are submitted to the browser process
      - renderer: `blink -> LayerTreeHost -> LayerTreeHostImpl -> ClientLayerTreeFrameSink`
      - browser: `RenderWidgetHostImpl -> RenderWidgetHostViewAura -> DelegatedFrameHost -> CompositorFrameSinkSupport`
  - the ui thread in the browser process runs the ui compositor and the
    display compositor
    - the ui compositor draws the ui
    - the display compositor composites renderer frames and the ui frames
    - when gpu compositing, it uses skia-over-gles where the gles impl is
      remote
    - the ui compositor submits `CompositorFrame` to
      `DirectLayerTreeFrameSink` which submits it to
      `CompositorFrameSinkSupport`
    - the display compositor has
      `HostFrameSinkManager`/`FrameSinkManagerImpl` and
      `GpuProcessTransportFactory`/`viz::Display`
- OOP-D, Out-Of-Process Display Compositor
  - the display compositor is moved from the ui thread in the browser
    process to a new `VizCompositorThread` thread in the gpu process
    - because the new thread and the main thread are both in the gpu
      process, a new `InProcessCommandBuffer` is introduced in place of
      `CommandBufferProxyImpl` and `GLES2CommandBufferStub`
    - `HostFrameSinkManager` is in the browser process and
      `FrameSinkManagerImpl` is in the gpu process
      - the ipc is `mojom::FrameSinkManager` and
        `mojom::FrameSinkManagerClient`
    - `VizProcessTransportFactory` is in the browser process and
      `RootCompositorFrameSinkImpl`/`viz::Display` is in the gpu process
      - the ipc is `mojom::DisplayPrivate`
  - the per-tab renderer process runs the renderer compositor
    - there is almost no change to the renderer compositor
    - except `CompositorFrame` is submitted to the gpu process directly
  - the ui thread in the browser process runs the ui compositor
    - `ClientLayerTreeFrameSink` is in the browser process and
      `RootCompositorFrameSinkImpl`/`CompositorFrameSinkSupport` is in the
      gpu process
      - the ipc is `mojom::CompositorFrameSink` and `mojom::CompositorFrameSinkClient`
- OOP-R, Out-Of-Process Rasterization
  - the motivation is skia-vk
  - the renderer process raster pipeline was
    - paint to `SkPicture`
    - raster task
      - skia-gl serialize to gles2 command buffer
    - gpu process deserializes and executes gles2 commands
  - with oopr, it becomes
    - pait to paint ops
      - `SkCanvas -> PaintCanvas`
      - `SkPaint -> PaintFlags`
      - `SkPicture -> PaintRecord/PaintOpBuffer`
      - `SkShader -> PaintShader`
      - `SkImage -> PaintImage`
    - raster task
      - serialize to raster command buffer
    - gpu processs deserializes and executes raster commands using skia-gl or
      skia-vk
  - `UseRasterDecoderForBrowserContext` feature
    - the browser uses raster command buffer rather than gles2 command buffer
      to draw the ui
- OOP-C, Out-Of-Process Rasterization for Canvas
  - OOP-R applies to the rasterization of non-canvas HTML elements
  - OOP-C applies to the rasterization of 2D canvas
- OOP-VD, Out-Of-Process Video Decode
  - OOP-D moves the display compositor from the browser process to the gpu process
  - OOP-R and OOP-C move rasterization from the renderer process to the gpu
    process
  - OOP-VD moves accelerated video decoding from the gpu process to the
    utility process

## Threads

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/threading_and_tasks.md>
- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/callback.md>
- browser process
  - the main thread ends up in `BrowserMainLoop::RunMainMessageLoop`, running
    `RunLoop`
  - the io thread is created by `BrowserMainLoop::CreateThreads`
  - the thread pool is created by `StartBrowserThreadPool`
- gpu process
  - the main thread ends up in `GpuMain`, running `RunLoop`
  - the io thread is created by `ChildProcess`
  - the thread pool is also created by `ChildProcess`
- task posting
  - a task is essentially a function call wrapped by `base::BindOnce`
  - `base::ThreadPool::CreateSingleThreadTaskRunner` or
    `base::SingleThreadTaskRunner::GetCurrentDefault` returns a single thread
    task runner
    - tasks posted to the single thread task runner are executed in-order on
      the same thread from the thread pool
  - `base::ThreadPool::CreateSequencedTaskRunner` or
    `base::SequencedTaskRunner::GetCurrentDefault` returns a sequenced task
    runner
    - tasks posted to the sequenced task runner are executed in-order, but may
      be on different threads from the thread pool
    - a "sequence" is a virtual thread
  - `base::ThreadPool::PostTask` posts a task to the thread pool
    - two tasks may be executed out-of-order
- physical threads
  - `base::SimpleThread` is an abstract class representing a physical thread
    - `SimpleThread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::SimpleThread::ThreadMain` which calls
      `base::SimpleThread::Run` virtual function
  - `base::Thread` is a concrete class representing a physical thread
    - `Thread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::Thread::ThreadMain` which enters
      `RunLoop::Run` to handle tasks

## Mojo

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/mojo_and_services.md>
- the most common way to create a message pipe between a client and a server
  is
  - the client creates a remote
    - `mojo::Remote<mojom::Foo> foo;`
  - the client creates a pending receiver
    - `mojo::PendingReceiver<mojom::Foo> receiver = foo.BindNewPipeAndPassReceiver();`
  - the client sends the pending receiver to the server
    - e.g., `BrowserInterfaceBroker::GetInterface`
    - `git grep 'GenericPendingReceiver' '*.mojom'`
  - the server provides an implementation
    - `class FooImpl : mojom::Foo { ... };`
    - the ctor converts the pending receiver to a receiver
  - the server registers a handler
    - `mojo::BinderMap::Add<mojom::Foo>(base::BindRepeating(createFoo, ...))`
    - when the server receives the pending receiver, it invokes the handler to
      create an implmentation object
- IO thread
  - browser process
    - `ContentMainRunnerImpl::RunBrowser` calls
      `BrowserTaskExecutor::CreateIOThread` to create the io thread and
      creates a `MojoIpcSupport` to handle mojo messages on the io thread
  - child process
    - `ChildProcess::ChildProcess` creates an `ChildIOThread`
    - `ChildThreadImpl::Init` calls `GetIOTaskRunner` to get the io thread
      task runner and create a `mojo::core::ScopedIPCSupport` to handle mojo
      messages on the io thread
- codegen
  - `FooStubDispatch` class is generated
  - the io thread calls `FooStubDispatch::Accept` with `Foo` and
    `mojo::Message`, and dispatches the message to `FooImpl->SomeMethod`
  - `FooImpl->SomeMethod` either executes the message on the io thread or
    schedules another task to free up io thread
- type maps
  - `BUILD.gn` has `cpp_typemaps` to map between mojo types and internal types
  - e.g., `media/mojo/mojom/BUILD.gn` says `media.mojom.VideoFrame` and
    `::scoped_refptr<::media::VideoFrame>` are convertible
  - the conversion uses `video_frame_mojom_traits.cc`'s `data` and `Read`
    methods

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
    - on a non-chrome dev build, `FIELDTRIAL_TESTING_ENABLED` is set by default
      - `VariationsFieldTrialCreatorBase::ApplyFieldTrialTestingConfig`
        applies the testing config generated from
        `fieldtrial_testing_config.json`
- flags
  - flags are defined in `chrome/browser/about_flags.cc`
  - each flag is associated with a switch or a feature
    - it provides a means to set switches/features persistently
    - not all switches/features have associated flags
- gpu
  - <https://chromium.googlesource.com/chromium/src/+/main/docs/gpu/debugging_gpu_related_code.md>
  - `--disable-gpu-sandbox`
  - `--no-sandbox`
  - `--enable-features=Vulkan,DefaultANGLEVulkan,VulkanFromANGLE`
    - `Vulkan` uses skia-vk rather than skia-gl
    - `DefaultANGLEVulkan` usea angle-vk rather than angle-gl (passthrough)
    - `VulkanFromANGLE` makes skia-vk and angle-vk share VkInstance and
      VkDevice
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

## WebUI

- <https://chromium.googlesource.com/chromium/src/+/main/docs/webui_explainer.md>
- `chrome:` protocol
  - `chrome://about`
  - `chrome://version`
  - `chrome://gpu`
  - `chrome://policy`
  - `chrome://flags`
  - more
- `chrome://gpu`
  - `RegisterContentWebUIConfigs` registers `GpuInternalsUIConfig` to handle
    `chrome://gpu`
  - when the url is requested, `config->CreateWebUIController` creates a
    `GpuInternalsUI`
  - `GpuInternalsUI` loads resources (html) from
    `content/browser/resources/gpu`

## Startup

- `chrome/app` defines the entrypoint
  - `chrome_exe_main_aura.cc` defines `main` and calls `ChromeMain`
  - `chrome_main.cc` defines `ChromeMain` and calls `ContentMain` with a
    `ChromeMainDelegate`
- `content/app` defines the app process
  - `content_main.cc` defines `ContentMain` and calls `RunContentProcess`
  - `RunContentProcess` calls `ContentMainRunner::{Initialize,Run}`
  - `content_main_runner_impl.cc` defines `ContentMainRunnerImpl::{Initialize,Run}`
    - `Initialize` calls `ChromeMainDelegate::PreSandboxStartup` which calls
      `crash_reporter::InitializeCrashpad` to fork `chrome_crashpad_handler`
    - because `process_type.empty()` is true, `Initialize` calls
      `InitializeZygoteSandboxForBrowserProcess` to fork two zygote processes
    - because `process_type.empty()` is true, `Run` calls `RunBrowser`,
      `RunBrowserProcessMain`, and `BrowserMain`
      - this process becomes the browser process
- `content/browser` defines the browser process
  - `browser_main.cc` defines `BrowserMain` and calls
    `BrowserMainRunner::{Initialize,Run}`
  - `browser_main_runner_impl.cc` defines
    `BrowserMainRunnerImpl::{Initialize,Run}`
    - `Initialize` calls `BrowserMainLoop::CreateStartupTasks` for various
      startup tasks
      - `GpuDataManagerImpl::GetInstance` initializes `GpuDataManagerImpl`
        which defines how to launch the gpu process
        - `InitializeGpuModes` initializes the possible modes
          - `gpu::GpuMode::HARDWARE_GL` is always on
          - `gpu::GpuMode::HARDWARE_VULKAN` is based on
            `ParseVulkanImplementationName`, which checks
            `features::IsUsingVulkan` (`--enable-features=Vulkan`)
      - `BrowserGpuChannelHostFactory::Initialize` establishes the gpu channel
        - `GpuProcessHost::Get` launches the gpu process by calling
          `LaunchGpuProcess`
          - `ChildProcessLauncherHelper::LaunchProcessOnLauncherThread` calls
            `ZygoteCommunication::ForkRequest` or `base::LaunchProcess` to
            fork the gpu process
          - because of `--type=gpu-process`, the gpu process's
            `ContentMainRunnerImpl::Run` calls `RunOtherNamedProcessTypeMain`
            which calls `GpuMain`
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

## GPU and WebGL

- WebKit expects the renderer to create a `WebKitClient`
- WebKit calls `WebKitClient::createGLES2Context` to create a `WebGLES2Context`
- the renderer implements `WebGLES2Context` with `ggl`, which in turns uses
  `gpu` for indirect rendering (renderer process to gpu process)
- the renderer sends commands to the gpu process (`chrome_gpu`)
  - the commands are handled by `GPUProcessor`
  - decoded GL commands are dispatched to `app/gfx/gl` directly
  - context management commands are sent to who?
- remote gles
  - initially, remote gles was used for everything
    - renderer/ui/display compositors all used remote gles, but they use
      remote raster now after OOP-D / OOP-R
    - webgl used remote gles and still does today
  - the client uses `GLES2Implementation`
    - it implements `GLES2Interface`, which includes
      `gles2_interface_autogen.h` to declare all GLES2 functions
    - it includes `gles2_implementation_autogen.h` and
      `gles2_implementation_impl_autogen.h` to implement all GLES2 functions
    - it uses `GLES2CmdHelper` to serialize the GLES2 function calls into a
      `CommandBufferProxyImpl`
  - the service in the gpu process uses `GLES2DecoderImpl` or
    `GLES2DecoderPassthroughImpl`
    - it uses `GLES2CommandBufferStub` to deserialize the commands and
      dispatches to `GLES2DecoderImpl::HandleFoo` or
      `GLES2DecoderPassthroughImpl::HandleFoo`
    - `GLES2DecoderImpl` validates before calling into the driver while
      `GLES2DecoderPassthroughImpl` calls into angle directly
- `SharedImageBacking`
  - webgl renders to a `SharedImageBacking` and compositor samples from the
    `SharedImageBacking`
  - by default on linux/x11, the image backing is `GLTextureImageBacking`
    - that is, webgl renders to a GL texture and compositor samples from it
    - `--enable-features=DefaultANGLEVulkan` uses `GLTextureImageBacking`
      - that is, everything works the same except angle translates gles to vk
        internally
    - `--enable-features=Vulkan` uses `ExternalVkImageBacking`
      - webgl renders to an imported vk image
      - compositor samples from the vk image
      - it crashes on radv and runs fine on anv
    - `--enable-features=Vulkan,DefaultANGLEVulkan` uses `OzoneImageBacking`
      - webgl renders to an imported gbm bo
      - compositor samples from the imported gbm bo
      - it gets tiling wrong on radv and runs fine on anv
    - `--enable-features=Vulkan,DefaultANGLEVulkan,VulkanFromANGLE` uses
      `AngleVulkanImageBacking`
      - webgl renders to a vk image
      - compositor samples from the vk image

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

## Perfetto

- Chrome uses TrackEvent data source in a different way
  - use perfetto ui, `Record new trace`, `Chrome` to record a trace
  - then go to `Info and stats` to see the exact config used
    - instead of `track_event` data source and `track_event_config`, it uses

      data_sources: {
        config {
          name: "org.chromium.trace_event"
          chrome_config {
              trace_config: "..."
          }
        }
      }
- <https://source.chromium.org/chromium/chromium/src/+/main:base/trace_event/builtin_categories.h>
  - `cc` is the compositor
    - <https://source.chromium.org/chromium/chromium/src/+/main:cc/>
    - in the renderer process, blink is the client
    - in the browser process, ui is the client
  - `disabled-by-default-skia.gpu` is skia gpu (ganesh and graphite)
    - <https://source.chromium.org/chromium/chromium/src/+/main:third_party/skia/src/gpu/>
  - `evdev` is ozone evdev (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/events/ozone/evdev/>
  - `drm` is ozone drm (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/drm/>
    - `hwoverlays` for overlay-related events
    - `drmcursor` for cursor-related events
  - `exo` is exo wayland compositor (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:components/exo/>
  - `input` is input
    - <https://source.chromium.org/chromium/chromium/src/+/main:content/browser/renderer_host/input/>
  - `mojom` is mojo ipc
    - <https://source.chromium.org/chromium/chromium/src/+/main:mojo/>
  - `gpu` is gpu-related events
    - <https://source.chromium.org/chromium/chromium/src/+/main:gpu/>
  - `gpu.angle` is angle
    - <https://source.chromium.org/chromium/chromium/src/+/main:third_party/angle/>
  - `ozone` is ozone abstraction layer
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/>
  - `toplevel` is top-level events
  - `toplevel.flow` is top-level flow events
  - `ui` is ui
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/>
  - `v8` is v8 js engine
    - <https://source.chromium.org/chromium/v8/v8>
  - `views` is ui views toolkit
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/views/>
  - `viz` is for composition and gpu presentation
    - <https://source.chromium.org/chromium/chromium/src/+/main:components/viz/>
  - `wayland` is ozone wayland
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/>
- `EnablePerfettoSystemTracing`
  - this feature is enabled by defaut for cros
  - `PerfettoTracedProcess::SetupSystemTracing` connects to the system
    perfetto socket
  - the gpu process cannot connect to the perfetto socket due to sandboxing,
    which can be disabled by `--disable-gpu-sandbox`

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

## Overview

- Front buffers
  - create "gbm_bo"
  - create drm fb using "drmModeAddFB2" from "gbm_bo" drm handle
  - create "EGLImage" from "gbm_bo" dmabufs
  - create gl fbos around "EGLImage"
- Swap chain
  - "drmModeSetCrtc"
  - GL draw and "glFinish"
  - "drmModePageFlip"

## GBM, Generic Buffer Manager

- "gbm_device" is a wrapper around DRM fd
  - usage limited to scanout, cursor, rendering, (CPU) write, and linear (tiling)
  - "is_format_supported", but usage is mostly ignored
  - "get_format_modifier_plane_count", and usage is omitted
  - "bo_create" with width/height/format/usage
  - "bo_create_with_modifiers" with modifiers but no usage
  - "bo_import" with usage
     - support WL_BUFFER, EGL_IMAGE, FD, and FD_PLANAR
     - buffer width/height/stride/format needs to be specified
  - "surface_create" with width/height/format/usage
  - "surface_create_with_modifiers" with modifiers but no usage
- "gbm_bo" is a buffer object
  - allocated or imported
  - point back to the "gbm_device"
  - width, height, format, modifier, bpp, stride, planar {strides,offsets} can be queried
  - "get_handle" to return platform-specific handle
  - "get_fd" to create a DMA-BUF (aka PRIME) fd, owned by the caller
  - "set_user_data" to assign a user data
  - "write" to write data into the BO
  - "map" and "unmap" for subregion mmaped access (might use a staging buffer)
- "gbm_surface" is a swap chain
  - it turns GBM into an EGL platform
  - there are initially N (>=1) free buffers
  - "has_free_buffers" must be called to make sure there are still free buffers
  - any EGL/GLES call probably requires and dequeues a free buffer
  - "eglSwapBuffers" queues the buffer back
  - "lock_front_buffer" acquires the queued buffer
  - "release_buffer" releases the acquired buffer
    - when N > 1, no need to release acquired buffer immediately each frame
- "gbm_dri.so" implementation
  - "gbm_device" internally consists of "__DRIscreen" and "__DRIcontext"
  - "gbm_bo" internally consists of "__DRIimage"
  - "gbm_surface" internally is implemented by EGL
    - that is why it is understood and can be used with EGL
- "minigbm" implementation
  - no "gbm_surface" support
  - only FD and FD-PLANAR imports
  - many more BO usage flags
  - some more BO queries (plane size, tiling, etc.)

## DRM/KMS

- https://dri.freedesktop.org/docs/drm/gpu/index.html
- echo 0x3f > /sys/module/drm/parameters/debug
- How to program DRM/KMS
  - drmModeGetResources
    - drmModeGetConnector
    - drmModeGetEncoder
    - to determine crtc
  - drmModeAddFB2 or drmModeAddFB2WithModifiers to add fb
    - for each potential scanout buffers
  - drmModeSetCrtc to set the crtc/fb/connector/mode
  - drmModePageFlip to flip fb
  - drmHandleEvent to wait for flip to finish

## EGL/GLES

- Platform extensions
  - EGL_MESA_platform_gbm
    - EGLNativeDisplayType is gbm_device
      - EGL_DEFAULT_DISPLAY works as well
    - EGLNativeWindowType is gbm_surface
  - EGL_MESA_platform_surfaceless
    - EGLNativeDisplayType must be EGL_DEFAULT_DISPLAY
    - No EGLNativeWindowType nor window surfaces
    - eglSwapBuffers only has effect on window surfaces by definition; it has
      no effect on this platform
- Context extensions
  - EGL_KHR_surfaceless_context
    - eglMakeCurrent without any surface
  - EGL_KHR_no_config_context
    - eglCreateContext without any EGLConfig
- Image extensions
  - to import DMA-BUFs as images for both sampling and rendering
  - EGL_KHR_image_base
  - EGL_EXT_image_dma_buf_import
  - EGL_EXT_image_dma_buf_import_modifiers
  - GL_OES_EGL_image
  - GL_OES_EGL_image_external
- Fence extensions
  - EGL_KHR_fence_sync
    - insert a sync object into the command stream
    - client-side waiting
  - EGL_KHR_wait_sync
    - server-side waiting
- drm-tests requires
  - EGL_KHR_surfaceless_context
  - DMA-BUF import as FBO

## Vulkan

- Memory extensions
  - allow VkDeviceMemory <-> dma_buf
  - VK_KHR_external_memory_capabilities
  - VK_KHR_external_memory
  - VK_KHR_external_memory_fd
  - VK_EXT_external_memory_dma_buf
- Sync extensions
  - VK_KHR_external_semaphore
  - VK_KHR_external_fence
- VK_EXT_image_drm_format_modifier
- VK_EXT_queue_family_foreign
- VK_ANDROID_external_memory_android_hardware_buffer
- https://github.com/SaschaWillems/Vulkan

## Tests

- drm-tests
  - https://chromium.googlesource.com/chromiumos/platform/drm-tests/

## Wayland

- XDG_RUNTIME_DIR=/var/run/chrome

## ML

- `services/on_device_model`
- `ChromeML::Get` loads the specified library or
  `optimization_guide_internal`, and finds the `GetChromeMLAPI` symbol

## Android

- features
  - `gpu/config/gpu_finch_features.cc`
    - `kVulkan` is `FEATURE_ENABLED_BY_DEFAULT`
  - `ui/gl/gl_switches.cc`
    - `kDefaultANGLEVulkan` is `FEATURE_DISABLED_BY_DEFAULT`
    - `kVulkanFromANGLE` is `FEATURE_DISABLED_BY_DEFAULT`

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
