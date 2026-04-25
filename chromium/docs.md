# Chromium Browser

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
