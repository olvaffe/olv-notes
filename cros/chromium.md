Chromium Browser
================

## Build system

* chromium uses `gclient` from `depot_tools` for source control
  * `gclient sync` to update the sources and `runhooks`
  * `gclient runhooks` to generate Makefiles from `*.gyp`
* GYP is a project for cross-platform building
  * `*.gyp` and `*.gypi`
* the top gyp file is `build/all.gyp`, which includes other `*.gyp` files
* `base/base.gyp`
  * `base` depends on `modp_b64`, `dynamic_annotations`, `allocator`,
    `symbolize`, `xdg_mime`, `libevent`, 
  * `base_i18n` depends on `base`, `icu18n`, `icuuc`
  * `symbolize` and `xdg_mime` are internal
  * `base/third_party/dynamic_annotations/dynamic_annotations.gyp`
  * `base/allocator/allocator.gyp`
  * `third_party/icu/icu.gyp`
  * `third_party/modp_b64/modp_b64.gyp`
  * `third_party/libevent/libevent.gyp`
* `net/net.gyp`
  * `net_base` depends on `base`, `base_i18n`, `googleurl`, `sdch`, `icui18n`,
    `icuuc`, `zlib`
  * `net` depends on `base`, `base_i18n`, `googleurl`, `sdch`, `icui18n`,
    `icuuc`, `zlib`, `net_base`
  * `sdch/sdch.gyp`
  * `build/temp_gyp/googleurl.gyp`
* `gfx/gfx.gyp`
  * `gfx` depends on `base`, `base_i18n`, `skia`, `icui18n`, `icuuc`,
    `libjpeg`, `libpng`, `sqlite`, `zlib`
  * `skia/skia.gyp`
  * `third_party/libjpeg/libjpeg.gyp`
  * `third_party/libpng/libpng.gyp`
  * `third_party/sqlite/sqlite.gyp`
  * `third_party/zlib/zlib.gyp`
* `app/app.gyp`
  * `app_base` depends on `app_resources`, `app_strings`, `base`,
    `base_i18n`, `gfx`, `net`, `skia`, `icu18n`, `icuuc`, `libjpeg`,
    `libpng`, `sqlite`, `zlib`
* `gpu/gpu.gyp`
  * for something like indirect rendering (because tabs are in different processes)
  * `command_buffer_common` depends on `base`
  * `command_buffer_client` depends on `command_buffer_common`
  * `gles2_cmd_helper` depends on `command_buffer_client`
  * `gles2_implementation` depends on `gles2_cmd_helper`
  * `gles2_lib` depends on `gles2_implementation`
  * `gles2_c_lib` depends on `gles2_lib`
  * `command_buffer_service` depends on `command_buffer_common`, `app_base`,
    `gfx`, `translator_glsl`
  * `gpu_plugin` depends on `base`, `command_buffer_service`
  * `third_party/angle/src/build_angle.gyp:translator_glsl`
* `chrome/chrome_common.gypi`
  * `common` depends on `chrome_resources`, `chrome_resources`,
    `chrome_strings`, `common_constants`, `common_net`, `default_plugin`,
    `theme_resources`, `app_base`, `app_resources`, `base`, `base_i18n`,
    `googleurl`, `ipc`, `net`, `printing`, `skia`, `bzip2`, `icui18n`, `icuuc`,
    `libxml`, `sqlite`, `zlib`, `npapi`, `appcache`, `blob`, `glue`
  * `common_net` depends on `chrome_resources`, `chrome_strings`, `app_base`,
    `base`, `net_resources`, `net`
  * `ipc/ipc.gyp`
  * `printing/printing.gyp`
  * `default_plugin/default_plugin.gyp:default_plugin`
  * `third_party/bzip2/bzip2.gyp:bzip2`
  * `third_party/libxml/libxml.gyp:libxml`
  * `third_party/npapi/npapi.gyp:npapi`
  * `webkit/support/webkit_support.gyp:appcache`
  * `webkit/support/webkit_support.gyp:blob`
  * `webkit/support/webkit_support.gyp:glue`
* `chrome/nacl.gypi`
  * native client
  * `nacl` depends on `chrome_resources`, `chrome_strings`, `common`, `npapi`,
    `glue`, `npGoogleNaClPluginChrome`, `sel`, `ncvalidate`, `platform_qual_lib`
  * `native_client/src/trusted/plugin/plugin.gyp:npGoogleNaClPluginChrome`
  * `native_client/src/trusted/service_runtime/service_runtime.gyp:sel`
  * `native_client/src/trusted/validator_x86/validator_x86.gyp:ncvalidate`
  * `native_client/src/trusted/platform_qualify/platform_qualify.gyp:platform_qual_lib`
* `third_party/WebKit/WebKit/chromium/WebKit.gyp`
  * THE WebKit
  * `webkit` depends on `webcore_bindings`, `googleurl`, `gles2_c_lib`, `icu*`,
    `libjpeg`, `libpng`, `libxml`, `libxslt`, `modp_64`, `nss`, `ots`, `zlib`, `v8`
  * `third_party/libxslt/libxslt.gyp`
  * `third_party/ots/ots.gyp`
* `chrome/chrome_renderer.gypi`
  * `renderer` depends on `common`, `common_net`, `plugin`, `chrome_resources`,
    `chrome_strings`, `printing`, `skia`, `hunspell`, `cld`, `ffmpeg`, `icui18n`,
    `icuuc`, `npapi`, `webkit`, `glue`, `webkit_resources`, `nacl`, `allocator`,
    `sandbox`, `gles2_c_lib`
  * `sandbox/sandbox.gyp`
  * `third_party/hunspell/hunspell.gyp:hunspell`
  * `third_party/cld/cld.gyp:cld`
  * `third_party/ffmpeg/ffmpeg.gyp:ffmpeg`
  * `third_party/WebKit/WebKit/chromium/WebKit.gyp:webkit`
* `media/media.gyp`
  * `media` depends on `base`, `ffmpeg`
  * `omx_wrapper` depends on `il`
  * `player_x11`
  * `third_party/ffmpeg/ffmpeg.gyp`
  * `third_party/openmax/openmax.gyp:il`
* `chrome/chrome_browser.gypi`
  * `browser` depends on `common`, `common_net`, `chrome_extra_resources`,
    `chrome_resources`, `chrome_strings`, `component_extensions`, `installer_util`,
    `platform_locale_settings`, `profile_import`, `views`,
    `browser/sync/protocol/sync_proto.gyp:sync_proto_cpp`, `syncapi`,
    `theme_resources`, `userfeedback_proto`, `app_resources`, `app_strings`,
    `media`, `printing`, `skia`, `bzip2`, `expat`, `icui18n`, `icuuc`, `libjingle`,
    `libxml`, `npapi`, `hunspell`, `libspeex`, `appcache`, `blob`, `database`,
    `glue`, `webkit_resources`,
  * `views/views.gyp`
* `chrome/chrome.gyp`
  * `chromium_dependencies` expands to `common`, `browser`, `debugger`,
    `chrome_gpu`, `profile_import`, `renderer`, `syncapi`, `utility`, `worker`,
    `service`, `printing`, `inspector_resources`
  * `chrome` depends on `(chromium_dependencies)`
  * `worker` depends on `base`, `skia`, `webkit` [Web Workers]
* `remoting/remoting.gyp`
  * for running apps in browser
  * `chromoting_host` depends on `chromoting_base`, `chromoting_jingle_glue`
  * `chromoting_client` depends on `chromoting_base`, `chromoting_jingle_glue`
  * `chromoting_plugin` depends on `chromoting_base`, `chromoting_client`,
    `chromoting_jingle_glue`, `ppapi_cpp_objects`
  * `chromoting_jingle_glue` depends on `notifier`, `libjingle`, `libjingle_p2p`
  * `chromoting_base` depends on `gfx`, `media`, `protobuf_lite`, `zlib`,
    `chromotocol_proto_lib`, `trace_proto_lib`, `chromoting_jingle_glue`
  * `third_party/protobuf2/protobuf.gyp:protobuf_lite`
  * `third_party/ppapi/ppapi.gyp:ppapi_cpp_objects`
  * `third_party/libjingle/libjingle.gyp`
  * `jingle/jingle.gyp:notifier`
  * `base/protocol/chromotocol.gyp`
* `chrome_frame/chrome_frame.gyp`
  * a plug-in for IE

## Components

* `base` is the most basic library
* `net` is the basic net library
* `app_base` has gfx, imaging, sqlite
* `chrome`
  * `browser` is the process running the UI and managing tabs and plugin
    processes
  * `renderer` is a process backing a single tab, based on `webkit`
  * `plugin` is a process backing a plugin
  * `service` is a process backing services such as cloud printing
  * `chrome_gpu` is a process that accepts 3D commands from renderers or NaCls
  * `nacl` stands for native client
  * `worker` based on `webkit` for [Web Workers]
  * `debugger`
* `npapi` is a common cross-browser plugin framework
* `ppapi` is an experimental plugin framework
* `libjingle` and `protobuf` are used to implement (XMPP-based) bookmark sync

## WebGL

* WebKit expects the renderer to create a `WebKitClient`
* WebKit calls `WebKitClient::createGLES2Context` to create a `WebGLES2Context`
* the renderer implements `WebGLES2Context` with `ggl`, which in turns uses
  `gpu` for indirect rendering (renderer process to gpu process)
* the renderer sends commands to the gpu process (`chrome_gpu`)
  * the commands are handled by `GPUProcessor`
  * decoded GL commands are dispatched to `app/gfx/gl` directly
  * context management commands are sent to who?

## GPU and IPC

* An `IPC::Channel` is created with a channel ID and consists of `socketpair`
  * A channel ID is a string of this form `"%d.%p.%d" % (pid, this, random())`

## Multi-process architecture

* every process is an instance of `chrome`, the executable/dll
* the type (browser/render/gpu/...) of a process is determined by its command
  line switch, `-type <type>`
