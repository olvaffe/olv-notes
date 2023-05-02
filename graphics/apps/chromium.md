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

## Command Line Options and Environment Variables

- edit `/etc/chrome_dev.conf`
- switches
  - `find -name '*_switches.cc'`
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
    - this is the default (set by the session manager) on a test build
    - otherwise, logging is redirected to `/home/chronos/user/log/chrome`
      after user login
- gpu
  - `--no-sandbox`
  - `--enable-features=Vulkan`

## `chrome://`

- `chrome://version`
- `chrome://gpu`
- `chrome://policy`
- `chrome://flags`

## Logs

- <https://chromium.googlesource.com/chromium/src/+/main/docs/chrome_os_logging.md>
- `/var/log/ui/ui.LATEST`
  - early stdout/stderr from chrome and session manager
- `/var/log/chrome/chrome`
  - chrome log after its logging subsystem has been initialized
- `base/logging.h`
  - `LOG(severity)` is enabled when `LOG_IS_ON(severity)` returns true
    - `--log-level=0` to enable `LOGGING_INFO`
  - `VLOG(verbosity)` is enabled when `VLOG_IS_ON(verbosity)` returns true
    - `--v=1` to enable verbosity 1
  - `DLOG` and `DVLOG` are enabled when `DCHECK_ALWAYS_ON` is enabled at
    compile time and the respective sevrity/verbosity is enabled at runtime

## Build system

- chromium uses `gclient` from `depot_tools` for source control
  - `gclient sync` to update the sources and `runhooks`
  - `gclient runhooks` to generate Makefiles from `*.gyp`
- GYP is a project for cross-platform building
  - `*.gyp` and `*.gypi`
- the top gyp file is `build/all.gyp`, which includes other `*.gyp` files
- `base/base.gyp`
  - `base` depends on `modp_b64`, `dynamic_annotations`, `allocator`,
    `symbolize`, `xdg_mime`, `libevent`, 
  - `base_i18n` depends on `base`, `icu18n`, `icuuc`
  - `symbolize` and `xdg_mime` are internal
  - `base/third_party/dynamic_annotations/dynamic_annotations.gyp`
  - `base/allocator/allocator.gyp`
  - `third_party/icu/icu.gyp`
  - `third_party/modp_b64/modp_b64.gyp`
  - `third_party/libevent/libevent.gyp`
- `net/net.gyp`
  - `net_base` depends on `base`, `base_i18n`, `googleurl`, `sdch`, `icui18n`,
    `icuuc`, `zlib`
  - `net` depends on `base`, `base_i18n`, `googleurl`, `sdch`, `icui18n`,
    `icuuc`, `zlib`, `net_base`
  - `sdch/sdch.gyp`
  - `build/temp_gyp/googleurl.gyp`
- `gfx/gfx.gyp`
  - `gfx` depends on `base`, `base_i18n`, `skia`, `icui18n`, `icuuc`,
    `libjpeg`, `libpng`, `sqlite`, `zlib`
  - `skia/skia.gyp`
  - `third_party/libjpeg/libjpeg.gyp`
  - `third_party/libpng/libpng.gyp`
  - `third_party/sqlite/sqlite.gyp`
  - `third_party/zlib/zlib.gyp`
- `app/app.gyp`
  - `app_base` depends on `app_resources`, `app_strings`, `base`,
    `base_i18n`, `gfx`, `net`, `skia`, `icu18n`, `icuuc`, `libjpeg`,
    `libpng`, `sqlite`, `zlib`
- `gpu/gpu.gyp`
  - for something like indirect rendering (because tabs are in different processes)
  - `command_buffer_common` depends on `base`
  - `command_buffer_client` depends on `command_buffer_common`
  - `gles2_cmd_helper` depends on `command_buffer_client`
  - `gles2_implementation` depends on `gles2_cmd_helper`
  - `gles2_lib` depends on `gles2_implementation`
  - `gles2_c_lib` depends on `gles2_lib`
  - `command_buffer_service` depends on `command_buffer_common`, `app_base`,
    `gfx`, `translator_glsl`
  - `gpu_plugin` depends on `base`, `command_buffer_service`
  - `third_party/angle/src/build_angle.gyp:translator_glsl`
- `chrome/chrome_common.gypi`
  - `common` depends on `chrome_resources`, `chrome_resources`,
    `chrome_strings`, `common_constants`, `common_net`, `default_plugin`,
    `theme_resources`, `app_base`, `app_resources`, `base`, `base_i18n`,
    `googleurl`, `ipc`, `net`, `printing`, `skia`, `bzip2`, `icui18n`, `icuuc`,
    `libxml`, `sqlite`, `zlib`, `npapi`, `appcache`, `blob`, `glue`
  - `common_net` depends on `chrome_resources`, `chrome_strings`, `app_base`,
    `base`, `net_resources`, `net`
  - `ipc/ipc.gyp`
  - `printing/printing.gyp`
  - `default_plugin/default_plugin.gyp:default_plugin`
  - `third_party/bzip2/bzip2.gyp:bzip2`
  - `third_party/libxml/libxml.gyp:libxml`
  - `third_party/npapi/npapi.gyp:npapi`
  - `webkit/support/webkit_support.gyp:appcache`
  - `webkit/support/webkit_support.gyp:blob`
  - `webkit/support/webkit_support.gyp:glue`
- `chrome/nacl.gypi`
  - native client
  - `nacl` depends on `chrome_resources`, `chrome_strings`, `common`, `npapi`,
    `glue`, `npGoogleNaClPluginChrome`, `sel`, `ncvalidate`, `platform_qual_lib`
  - `native_client/src/trusted/plugin/plugin.gyp:npGoogleNaClPluginChrome`
  - `native_client/src/trusted/service_runtime/service_runtime.gyp:sel`
  - `native_client/src/trusted/validator_x86/validator_x86.gyp:ncvalidate`
  - `native_client/src/trusted/platform_qualify/platform_qualify.gyp:platform_qual_lib`
- `third_party/WebKit/WebKit/chromium/WebKit.gyp`
  - THE WebKit
  - `webkit` depends on `webcore_bindings`, `googleurl`, `gles2_c_lib`, `icu*`,
    `libjpeg`, `libpng`, `libxml`, `libxslt`, `modp_64`, `nss`, `ots`, `zlib`, `v8`
  - `third_party/libxslt/libxslt.gyp`
  - `third_party/ots/ots.gyp`
- `chrome/chrome_renderer.gypi`
  - `renderer` depends on `common`, `common_net`, `plugin`, `chrome_resources`,
    `chrome_strings`, `printing`, `skia`, `hunspell`, `cld`, `ffmpeg`, `icui18n`,
    `icuuc`, `npapi`, `webkit`, `glue`, `webkit_resources`, `nacl`, `allocator`,
    `sandbox`, `gles2_c_lib`
  - `sandbox/sandbox.gyp`
  - `third_party/hunspell/hunspell.gyp:hunspell`
  - `third_party/cld/cld.gyp:cld`
  - `third_party/ffmpeg/ffmpeg.gyp:ffmpeg`
  - `third_party/WebKit/WebKit/chromium/WebKit.gyp:webkit`
- `media/media.gyp`
  - `media` depends on `base`, `ffmpeg`
  - `omx_wrapper` depends on `il`
  - `player_x11`
  - `third_party/ffmpeg/ffmpeg.gyp`
  - `third_party/openmax/openmax.gyp:il`
- `chrome/chrome_browser.gypi`
  - `browser` depends on `common`, `common_net`, `chrome_extra_resources`,
    `chrome_resources`, `chrome_strings`, `component_extensions`, `installer_util`,
    `platform_locale_settings`, `profile_import`, `views`,
    `browser/sync/protocol/sync_proto.gyp:sync_proto_cpp`, `syncapi`,
    `theme_resources`, `userfeedback_proto`, `app_resources`, `app_strings`,
    `media`, `printing`, `skia`, `bzip2`, `expat`, `icui18n`, `icuuc`, `libjingle`,
    `libxml`, `npapi`, `hunspell`, `libspeex`, `appcache`, `blob`, `database`,
    `glue`, `webkit_resources`,
  - `views/views.gyp`
- `chrome/chrome.gyp`
  - `chromium_dependencies` expands to `common`, `browser`, `debugger`,
    `chrome_gpu`, `profile_import`, `renderer`, `syncapi`, `utility`, `worker`,
    `service`, `printing`, `inspector_resources`
  - `chrome` depends on `(chromium_dependencies)`
  - `worker` depends on `base`, `skia`, `webkit` [Web Workers]
- `remoting/remoting.gyp`
  - for running apps in browser
  - `chromoting_host` depends on `chromoting_base`, `chromoting_jingle_glue`
  - `chromoting_client` depends on `chromoting_base`, `chromoting_jingle_glue`
  - `chromoting_plugin` depends on `chromoting_base`, `chromoting_client`,
    `chromoting_jingle_glue`, `ppapi_cpp_objects`
  - `chromoting_jingle_glue` depends on `notifier`, `libjingle`, `libjingle_p2p`
  - `chromoting_base` depends on `gfx`, `media`, `protobuf_lite`, `zlib`,
    `chromotocol_proto_lib`, `trace_proto_lib`, `chromoting_jingle_glue`
  - `third_party/protobuf2/protobuf.gyp:protobuf_lite`
  - `third_party/ppapi/ppapi.gyp:ppapi_cpp_objects`
  - `third_party/libjingle/libjingle.gyp`
  - `jingle/jingle.gyp:notifier`
  - `base/protocol/chromotocol.gyp`
- `chrome_frame/chrome_frame.gyp`
  - a plug-in for IE

## Components

- `base` is the most basic library
- `net` is the basic net library
- `app_base` has gfx, imaging, sqlite
- `chrome`
  - `browser` is the process running the UI and managing tabs and plugin
    processes
  - `renderer` is a process backing a single tab, based on `webkit`
  - `plugin` is a process backing a plugin
  - `service` is a process backing services such as cloud printing
  - `chrome_gpu` is a process that accepts 3D commands from renderers or NaCls
  - `nacl` stands for native client
  - `worker` based on `webkit` for [Web Workers]
  - `debugger`
- `npapi` is a common cross-browser plugin framework
- `ppapi` is an experimental plugin framework
- `libjingle` and `protobuf` are used to implement (XMPP-based) bookmark sync

## WebGL

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

## Multi-process architecture

- every process is an instance of `chrome`, the executable/dll
- the type (browser/render/gpu/...) of a process is determined by its command
  line switch, `-type <type>`

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

## GPU Memory

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

## Vulkan

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
