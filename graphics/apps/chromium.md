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

## MotionMark

- <https://github.com/WebKit/MotionMark>
  - `python -m http.server -d MotionMark` and open
    <http://localhost:8000/index.html>
- `window.addEventListener("load", ...)`
  - `window.sectionsManager` is `SectionsManager`
  - `window.benchmarkController` is `BenchmarkController`
  - `BenchmarkController::initialize` detects the framerate
    - it disables the start button
    - `detectFrameRate`
      - `requestAnimationFrame` calls `tick` on next repaint
      - it measures the time needed for 300 repaints
      - `finish` returns the closest well-known framerate, typically 60
    - `frameRateDeterminationComplete` shows the detected framerate and
      enables the start button
- `<button id="start-button" onclick="benchmarkController.startBenchmark()">`
  calls `BenchmarkController::startBenchmark`
  - `determineCanvasSize` adds a class to `<body>` depending on the screen
    width
    - `window.matchMedia` queries the screen width
      - <https://en.wikipedia.org/wiki/Media_queries> 
    - `document.body.classList.add` adds the class
    - CSS picks the size for `frame-container`
      - `small` is 568x320
      - `medium` is 900x600
      - `large` is 1600x800
  - `benchmarkDefaultParameters` specifies the default test options
    - open `developer.html` instead to customize the options
  - `Suites` contains a single `Suite` (unless in `developer.html`)
    - the `MotionMark` suite consists of several tests
      - `Multiply` uses `core/multiply.html`
      - `Canvas Arcs` uses `core/canvas-stage.html?pathType=arcs`
      - `Leaves` uses `core/leaves.html`
      - `Paths` uses `core/canvas-stage.html?pathType=linePath`
      - `Canvas Lines` uses `core/canvas-stage.html?pathType=line&lineCap=square`
      - `Images` uses `core/image-data.html`
      - `Design` uses `core/design.html`
      - `Suits` uses `core/suits.html`
  - `_startBenchmark`
    - `ensureRunnerClient` creates `BenchmarkRunnerClient`
    - `BenchmarkRunner::runMultipleIterations` runs the tests
      - `BenchmarkRunnerClient::willStartFirstIteration` creates
        `ResultsDashboard`
      - `runAllSteps` calls `step`
        - first step creates a `BenchmarkRunnerState`
          - this is an iterator pointint to the first test of first suite
        - first test calls `_appendFrame`
          - this inserts an `<iframe>` into `<section id="test-container">`
        - `prepareCurrentTest` points the `<iframe>` to the test url and calls
          `_runBenchmarkAndRecordResults`
          - it creates `contentWindow.benchmarkClass` test class and runs it
    - `SectionsManager::showSection`
- `Benchmark::run`
- `MultiplyBenchmark` is a `Benchmark` with `MultiplyStage`

## Life of a Pixel/Frame

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/life_of_a_frame.md>
- <https://bit.ly/lifeofapixel>
- when user enters a url, browser process
  - `CrBrowserMain` calls `OpenCurrentURL` to open the url
  - this starts a navigation flow
    - `NavigationRequest::NavigationRequest`
    - `NavigationRequest::BeginNavigation`
    - `NavigationRequest::StartNavigation`
    - `NavigationRequest::OnResponseStarted`
      - `RenderFrameHostManager::CreateSpeculativeRenderFrame` calls
        `RenderFrameHostImpl::CreateRenderFrame`
    - `NavigationRequest::CommitNavigation`
      - `RenderFrameHostImpl::CommitNavigation`
- in response to `RenderFrameHostImpl::CreateRenderFrame`, renderer process
  - `CrRendererMain` calls `AgentSchedulingGroup::CreateFrame`
  - this starts a loading flow
    - `DocumentLoader::DocumentLoader`
    - `DocumentLoader::CommitNavigation`
    - `DocumentLoader::StartLoadingResponse`
    - `DocumentLoader::FinishedLoading`
- in response to `RenderFrameHostImpl::CommitNavigation`, renderer process
  - `CrRendererMain` calls `RenderFrameImpl::CommitNavigation`
  - `HTMLDocumentParser` and `HTMLTreeBuilder` parses html to DOM
  - `CSSParser` styles DOM
  - `HTMLParserScriptRunner` execs js
- when the compositor needs a new frame (e.g., in response to vsync), it asks
  clients (e.g., tabs, canvass, browser ui, etc.) to submit new
  `CompositorFrame`s
  - gpu `VizCompositorThread` calls `DisplayScheduler::OnBeginFrame`
  - renderer `Compositor` calls `ExternalBeginFrameSource::OnBeginFrame`
    - when the client is a tab
  - renderer `CrRendererMain` calls `ProxyMain::BeginMainFrame`
    - `LocalFrameView::RunStyleAndLayoutLifecyclePhases` updates dom style and layout
    - `LocalFrameView::RunPrePaintLifecyclePhase` prepaints
    - `LocalFrameView::RunPaintLifecyclePhase` paints
      - this converts layout tree to display items / paint ops
    - `LayerTreeHost::DoUpdateLayers` updates layers
      - this converts layout tree to layer list
    - `ProxyMain::BeginMainFrame::commit`
  - renderer `Compositor` calls `ProxyImpl::ReadyToCommit`
    - `LayerTreeHostImpl::BeginCommit`
    - `LayerTreeHostImpl::FinishCommit`
    - `LayerTreeHostImpl::CommitComplete`
      - `LayerTreeImpl::UpdateTiles`
      - `TileManager::PrepareTiles`
        - `TileManager::AssignGpuMemoryToTiles`
        - `TileManager::ScheduleTasks`
          - this divides layers into tiles and `RasterTask`s
          - `GpuRasterBuffer::Playback` requests the gpu process to rasterize
            `RasterTask`s
    - `TileManager::CheckForCompletedTasksAndIssueSignals`
      - `LayerTreeHostImpl::ActivateSyncTree` activates the latest rasterized
        layers (they are double-buffered) for drawing
    - `ProxyImpl::ScheduledActionDraw` draws layers
      - `LayerTreeHostImpl::PrepareToDraw`
      - `LayerTreeHostImpl::DrawLayers`
        - `LayerTreeHostImpl::GenerateCompositorFrame`
        - `GpuChannel::Flush`
        - `LayerContextImpl::SubmitCompositorFrame`
  - gpu `VizCompositorThread` calls
    `CompositorFrameSinkImpl::SubmitCompositorFrame`
    - `Surface::QueueFrame`
    - `Surface::ActivateFrame`
- when the compositor presents a new frame (e.g., also in response to vsync)
  - gpu `VizCompositorThread` calls `DisplayScheduler::OnBeginFrameDeadline`
    - `DisplayScheduler::DrawAndSwap` calls `Display::DrawAndSwap`
    - `SurfaceAggregator::Aggregate` aggregates all `CompositorFrame`s from
      all clients to an `AggregatedFrame`
    - `DirectRenderer::DrawFrame` draws the `AggregatedFrame`
      - this draws with `SkiaRenderer` and draws to `GrDeferredDisplayList`
    - `SkiaRenderer::SwapBuffers`
      - `SkiaOutputSurfaceImpl::SwapBuffers`
  - gpu `CrGpuMain`
    - `SkiaOutputSurfaceImplOnGpu::FinishPaintRenderPass`
      - this replays `GrDeferredDisplayList`
    - `SkiaOutputSurfaceImplOnGpu::SwapBuffers`
      - `SkiaOutputDevice::Submit` calls `GrDirectContext::submit` to submit
        (`glFlush` or `vkQueueSubmit`) cmdbufs to gpu
      - `SkiaOutputSurfaceImplOnGpu::PostSubmit` calls
        `SkiaOutputSurfaceImplOnGpu::PresentFrame`
        - `SkiaOutputDeviceBufferQueue::ScheduleOverlays`
        - `SkiaOutputDeviceBufferQueue::Present`
          - on wayland, this calls `GbmSurfacelessWayland::Present`
          - on drm, this calls `GbmSurfaceless::Present`
