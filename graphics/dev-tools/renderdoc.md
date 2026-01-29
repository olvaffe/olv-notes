RenderDoc
=========

## Build

- `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DENABLE_QRENDERDOC=off -DENABLE_PYRENDERDOC=off`
  - to enable `qrenderdoc`,
    - `apt install python3-dev qtbase5-dev libqt5svg5-dev libqt5x11extras5-dev`
  - to disable GL/GLES,
    - `-DENABLE_GL=OFF`, `-DENABLE_GLES=OFF`, and `-DENABLE_EGL=OFF`
- `ninja install`, or
  - edit `renderdoc_capture.json` to point to local `librenderdoc.so`
  - copy `renderdoc_capture.json` to `~/.local/share/vulkan/implicit_layer.d`

## Cross-Compile

- THIS DOES NOT WORK.  USE REMOTE SERVER INSTEAD.
- build qt
  - `wget https://download.qt.io/archive/qt/5.15/5.15.3/single/qt-everywhere-opensource-src-5.15.3.tar.xz`
  - `PKG_CONFIG_LIBDIR=... ../configure -extprefix `pwd`/aaa -opensource
       -confirm-license -release -static -xplatform linux-aarch64-gnu-g++
       -sysroot <sysroot> -opengl es2 -xcb-xlib -xcb -nomake tests
       -nomake examples -nomake tools -skip qtwebengine -skip qtwebglplugin`
    - look for `xcb_syslibs` in `qtbase/src/gui/configure.json`, which hints
      the dependencies for x11extras
    - I need to install dependencies in both host and sysroot.  Something is
      not right.
  - `make` and `make install`
- configure renderdoc with `-DSTATIC_QRENDERDOC=on` and
  `-DQT_QMAKE_EXECUTABLE=<path-to-qmake>`
  - edit `qrenderdoc/CMakeLists.txt` such that `SWIG_CONFIGURE_CC` and
    `SWIG_CONFIGURE_CXX` use the host compilers

## Android

- `ANDROID_HOME=~/android/sdk ANDROID_NDK=~/android/sdk/ndk/28.2.13676358 JAVA_HOME=/usr/lib/jvm/default-java \
   cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DBUILD_ANDROID=On -DANDROID_ABI=arm64-v8a \
   -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
- build fixes
  - `s/ALooper_pollAll/ALooper_pollOnce/`
  - `s/-source 1.7 -target 1.7/-source 1.8 -target 1.8/`
  - `s/-Wno-cast-function-type-mismatch/-Wno-cast-function-type-strict/`
- `adb push out/lib/libVkLayer_GLES_RenderDoc.so /data/local/debug/vulkan`
- cmake internals
  - with `BUILD_ANDROID`,
    - `JAVA_HOME` must be defined and `$JAVA_HOME/bin/java` is the bin
    - `ANDROID_HOME` must be defined and `$ANDROID_HOME/build-tools` must
      exist
    - `ANDROID_NDK` must be defined and
      `$ANDROID_NDK/build/cmake/android.toolchain.cmake` must exist
    - `ANDROID_PLATFORM` defaults to `android-21`
    - `ANDROID_STL` defaults to `c++_static`
    - `ANDROID_ABI` defaults to `armeabi-v7a`
  - `libVkLayer_GLES_RenderDoc.so`
    - it is renamed from `librenderdoc.so` by setting `OUTPUT_NAME`
  - `librenderdoccmd.so`
    - it is `renderdoccmd` as a native activity
  - apk
    - auto-selects the latest `$ANDROID_HOME/build-tools` version
    - auto-selects the latest `$ANDROID_HOME/platforms` version
    - generates `debug.keystore` using java `keytool`
    - depends on the two shared libs, and copys them to `libs/lib/<abi>`
    - `aapt package ...` generates `R.java`
    - `javac ...` compiles `*.java` to `*.class`
    - `d8 ...` compiles `*.class` to `classes.dex`
    - `aapt package -F RenderDocCmd-unaligned.apk ...` packages `RenderDocCmd-unaligned.apk`
    - `zipalign RenderDocCmd-unaligned.apk RenderDocCmd.apk` generates `RenderDocCmd.apk`
    - `apksigner sign ...` signs `RenderDocCmd.apk` using `debug.keystore`

## `renderdoccmd`

- capture
  - `renderdoccmd capture -c <output-name> -w <executable> ...`
  - might want `-d` to specify the working dir
- replay
  - `renderdoccmd replay <capture.rdc>` just works
  - `renderdoccmd remoteserver` just works for remote replay
- internals
  - init
    - `REPLAY_PROGRAM_MARKER` defines a magic symbol
    - upon `librenderdoc.so` loading, `library_loaded` calls
      `RenderDoc::Initialise` and marks itself as replaying
    - `RENDERDOC_InitialiseReplay` inits for replay
  - `capture` calls `RENDERDOC_ExecuteAndInject`
  - `replay` calls
    - if locally, `RENDERDOC_OpenCaptureFile` and `CaptureFile::OpenCapture`
    - if remotely, `RENDERDOC_CreateRemoteServerConnection`,
      `RemoteServer::CopyCaptureToRemote`, and `RemoteServer::OpenCapture`
    - `ReplayController::CreateOutput`
    - `ReplayOutput::Display`
  - `remoteserver` calls `RENDERDOC_BecomeRemoteServer`

## Directory Structure

- `qrenderdoc` is the GUI frontend
- `renderdoc` provides `librenderdoc.so`
  - `android` provides android support for frontends
    - it registers a `AndroidController` automatically to enumerate and
      control android devices using adb
  - `api/app` provides public headers for apps
  - `api/replay` provides public headers for frontends
  - `common` provides common helpers
  - `core` provides renderdoc core
  - `driver/gl` provides gl support for both apps and frontends
  - `driver/vulkan` provides vulkan support for both apps and frontends
  - `hooks` manages driver hooks
    - upon dlopen, `LibraryHooks::RegisterHooks` registers driver hooks
  - `os` is os-specific helpers
  - `replay` provides local replay support
  - `serialise` serializes/deserializes data types
    - depending on whether the serializer is `WriteSerialiser` or
      `ReadSerialiser`, `SERIALISE_ELEMENT` serializes or deserializes an
      element respectively
- `renderdoccmd` is the CLI frontend

## Logging

- `RDCLOG` logs a `LogType::Comment`
  - `RDCDEBUG` logs a `LogType::Debug` but copmiles to nop by default
- `rdclog_direct` logs to a buffer
- `rdclogprint_int` prints the buffer
  - it prints to `OSUtility::Output_DebugMon`, which is logcat on android
  - it prints to log file
    - `GetDefaultFiles`
      - `/tmp` or `$RENDERDOC_TEMP` is the temp dir
      - `/tmp/RenderDoc/*.log` is the log file
- upon exit, `RDCSTOPLOGGING` removes the log file

## Performance Counter

- `Capture counters` calls `PerformanceCounterViewer::CaptureCounters`
  - `PerformanceCounterSelection` dialog is shown
  - `GLReplay::EnumerateCounters` or `VulkanReplay::EnumerateCounters`
    enumerates counters
    - for gl, `ARB_timer_query` (or `EXT_disjoint_timer_query`) maps to
      `GPUCounter::EventGPUDuration`
  - it blocks until `Sample counters` is clicked to return `QDialog::Accepted`
  - `GLReplay::FetchCounters` or `VulkanReplay::FetchCounters` replays with
    the specified counters
- `Time durations for the actions` calls
  `EventBrowser::on_timeActions_clicked`
  - the timings are collected with
    `r->FetchCounters({GPUCounter::EventGPUDuration})`

## Remote Context

- `DeviceProtocolRegistration` registers a dev proto
  - adb is the only dev proto
- `MainWindow::MainWindow` spawns a thread to call `remoteProbe`
  - `UpdateEnumeratedProtocolDevices` calls `AndroidController::GetDevices`
    - `Android::EnumerateDevices` invokes `adb devices`
    - `Android::IsSupported` invokes `adb shell getprop ro.build.version.sdk`
    - `Android::adbForwardPorts` invokes `adb forward tcp:<port> localabstract:<socket>`
      - this forwards local tcp port to remote unix socket
  - `RemoteHost::CheckStatus` checks the remote status by connecting to the
    remote server and querying the remote server version
- when a remote is selected, `MainWindow::setRemoteHost` is called
  - `RemoteHost::Launch` calls `AndroidController::StartRemoteServer`
    - `Android::ListPackages` invokes `adb shell pm list packages --user $(am get-current-user) org.renderdoc.renderdoccmd`
    - `Android::GetSupportedABIs` invokes `adb shell getprop ro.product.cpu.abi`
    - `Android::RemoveRenderDocAndroidServer` invokes `adb uninstall ...` to
      uninstall outdated apks
    - `Android::InstallRenderDocServer` invokes `adb install -r -g --force-queryable <renderdoc>/share/renderdoc/plugins/android/org.renderdoc.renderdoccmd.<abi>.apk`
      - `--force-queryable` forces the app queryable by other apps
    - `adb push $HOME/.renderdoc/renderdoc.conf /sdcard/Android/media/org.renderdoc.renderdoccmd.<abi>/files`
    - `adb shell am start -n org.renderdoc.renderdoccmd.<abi>/.Loader -e renderdoccmd remoteserver`
  - `ReplayManager::ConnectToRemoteServer` connects to the remote server
    - thanks to adb port forwarding, this connects to a local tcp port
- apk `Loader` class inherits from `android.app.NativeActivity`
  - it requests `android.permission.MANAGE_EXTERNAL_STORAGE`
  - `android_main` is the native entrypoint
  - `getRenderdoccmdArgs` gets the args, which are `renderdoccmd remoteserver`
  - `DisplayGenericSplash` displays the splash using egl/gles
  - `RenderDoc::BecomeRemoteServer` enters the main loop
    - on android, it listens to a adb-forwarded socket
- when we select an executable to capture,
  - `ReplayManager::ListFolder` calls `AndroidRemoteServer::ListFolder`
    - `adb shell pm list packages --user $(am get-current-user) -3` lists
      3rd-party packages
    - `adb shell dumpsys packages` lists package activities
  - `CaptureDialog::TriggerCapture` calls `MainWindow::OnCaptureTrigger`
    - `ReplayManager::ExecuteAndInject` calls `AndroidRemoteServer::ExecuteAndInject`
      - `adb shell settings put global enable_gpu_debug_layers 1`
      - `adb shell settings put global gpu_debug_app <package>`
      - `adb shell settings put global gpu_debug_layer_app org.renderdoc.renderdoccmd.<abi>`
      - `adb shell settings put global gpu_debug_layers VK_LAYER_RENDERDOC_Capture`
      - `adb shell settings put global gpu_debug_layers_gles libVkLayer_GLES_RenderDoc.so`
      - `adb shell mkdir -p /sdcard/Android/media/<package>/files`
      - `adb shell setprop debug.rdoc.RENDERDOC_CAPOPTS <opts>`
      - `adb push $HOME/.renderdoc/renderdoc.conf /sdcard/Android/media/<package>/files`
      - `adb shell am start -S -n <package>/<activity>`
    - `MainWindow::ShowLiveCapture` shows the live capture tab
      - `RENDERDOC_CreateTargetControl` connects to the app
- when we trigger a capture, `LiveCapture::on_triggerImmediateCapture_clicked`
  - it allows `LiveCapture::connectionThreadEntry` to call
    `TargetControl::TriggerCapture` to send `ePacket_TriggerCapture`
  - app `RenderDoc::TargetControlClientThread` handles the packet and calls
    `RenderDoc::TriggerCapture`
  - on next frame, app `RenderDoc::ShouldTriggerCapture` returns true and
    - `WrappedVulkan::StartFrameCapture`
    - `WrappedVulkan::EndFrameCapture`
      - `RenderDoc::CreateRDC` creates a `RDCFile` for capture serialization
      - `RenderDoc::FinishCaptureWriting` writes the `RDCFile` to disk

## `VK_LAYER_RENDERDOC_Capture`

- `library_loaded` is called upon `dlopen`
  - `RenderDoc::Initialise`
    - `Network::CreateServerSocket` listens on port
      `RenderDoc_FirstTargetControlPort`
    - `FileIO::GetDefaultFiles` determines output filenames
      - on android, `GetTempRootPath` returns `/sdcard/Android/media/<package>/files`
  - `LibraryHooks::RegisterHooks` registers hooks for various apis
    - `EGLHook::RegisterHooks` early returns on android because of
      `EGL_ANDROID_GLES_layers`
    - `GLHook::RegisterHooks` early returns on android similarly
    - `VulkanHook::RegisterHooks` inits itself
- loader finds GIPA/GDPA
  - `VK_LAYER_RENDERDOC_CaptureGetInstanceProcAddr`
  - `VK_LAYER_RENDERDOC_CaptureGetDeviceProcAddr`
- app calls `vkCreateInstance` and ends up in `hooked_vkCreateInstance`
  - `KeepLayerAlive` creates an (ununsed) internal instance on android
  - `WrappedVulkan::vkCreateInstance`
    - `RenderDoc::AddDeviceFrameCapturer` creates a capturer
- vk 1.0 app failure with renderdoc capture
  - logcat has
    - `vulkan  : missing dev proc: vkGetDeviceGroupPresentCapabilitiesKHR`
    - `vulkan  : missing dev proc: vkGetDeviceGroupSurfacePresentModesKHR`
    - `vulkan  : missing dev proc: vkAcquireNextImage2KHR`
  - this is because renderdoc does not return those functions unless vk 1.1 or
    `VK_KHR_device_group` is enabled

## GLES Capture

- `library_loaded` is called upon `dlopen`
- on android, loader finds
  - `AndroidGLESLayer_Initialize` inits the layer
    - `EGL` and `GL` dispatch tables are initialized with func pionters from
      driver or the next layer
  - `AndroidGLESLayer_GetProcAddress` returns func pointers to loader or prev
    layer
    - `eglFoo_renderdoc_hooked`
    - `glFoo_renderdoc_hooked` defined via `DefineSupportedHooks`
- app calls `eglCreateContext` and ends up in `eglCreateContext_renderdoc_hooked`
  - `WrappedOpenGL::CreateContext`
    - `RenderDoc::AddDeviceFrameCapturer` creates a capturer
- app calls `eglMakeCurrent` and ends up in `eglMakeCurrent_renderdoc_hooked`
  - `WrappedOpenGL::ActivateContext`

## Capture Example

- `VK_IMPLICIT_LAYER_PATH=~/projects/renderdoc/out/lib ENABLE_VULKAN_RENDERDOC_CAPTURE=1 vkcube --wsi xcb`
  - it must be an implicit layer to intercept pre-instance functions
- `CaptureState::BackgroundCapturing` is the initial state
  - app `vkCreateBuffer`
    - `hooked_vkCreateBuffer` is defined by `HookDefine4(VkResult, vkCreateBuffer, ...)`
    - `WrappedVulkan::vkCreateBuffer`
      - it calls down to the driver with modified create info
      - `VulkanResourceManager::WrapResource` wraps the driver buf
      - `VulkanResourceManager::AddResourceRecord` adds associated `VkResourceRecord`
      - `Serialise_vkCreateBuffer` serializes the call to a chunk
      - `ResourceRecord::AddChunk` adds the chunk to the record
  - app `vkAllocateMemory`
    - `hooked_vkAllocateMemory` is defined by `HookDefine4(VkResult, vkAllocateMemory, ...)`
    - `WrappedVulkan::vkAllocateMemory`
      - it calls down to the driver with modified alloc info
      - `VulkanResourceManager::WrapResource` wraps the driver mem
      - `VulkanResourceManager::AddResourceRecord` adds associated `VkResourceRecord`
      - `Serialise_vkAllocateMemory` serializes the call to a chunk
      - `ResourceRecord::AddChunk` adds the chunk to the record
      - `VulkanResourceManager::AddDeviceMemory` tracks the mem
  - app `vkBindBufferMemory` is pretty regular
  - app `vkMapMemory`
    - `hooked_vkMapMemory` is defined by `HookDefine6(VkResult, vkMapMemory, ...)`
    - `WrappedVulkan::vkMapMemory`
      - it calls down to the driver
      - it adds the addr to the record
  - app `vkQueuePresentKHR`
    - `hooked_vkQueuePresentKHR` is defined by `HookDefine2(VkResult, vkQueuePresentKHR, ...)`
    - `WrappedVulkan::vkQueuePresentKHR`
      - `HandlePresent` draws the overlay
        - `RenderDoc::GetOverlayText` returns the overlay text
      - it calls down to the driver
      - no `Serialise_vkQueuePresentKHR`
      - `WrappedVulkan::Present`
- when capture is triggered,
  - app `vkQueuePresentKHR`
    - `WrappedVulkan::vkQueuePresentKHR`
      - `WrappedVulkan::Present`
        - `RenderDoc::ShouldTriggerCapture` returns true when
          - `RenderDoc::TriggerCapture` has incremented `m_Cap`
          - `RenderDoc::QueueCapture` has added a frame to `m_QueuedFrameCaptures`
        - `RenderDoc::StartFrameCapture` starts the capture
          - `WrappedVulkan::StartFrameCapture`
            - it sets state to `CaptureState::ActiveCapturing`
- `CaptureState::ActiveCapturing` is the active state
  - app `vkQueuePresentKHR`
    - `WrappedVulkan::vkQueuePresentKHR`
      - because we are active, `Serialise_vkQueuePresentKHR` serializes the
        call to a chunk
      - `WrappedVulkan::Present`
        - because we are active, `RenderDoc::EndFrameCapture` ends the capture
          - `WrappedVulkan::EndFrameCapture`
            - `EndCaptureFrame` serializes the backbuffer to a chunk
            - it sets the state back to `CaptureState::BackgroundCapturing`
            - it reads back the backbuffer for screenshot
            - `RenderDoc::CreateRDC` creates an rdc
            - it serializes all chunks referenced by the frame to the rdc

## Replay Example

- `renderdoccmd replay vkcube.rdc`
- `CaptureFile::OpenFile` calls `RDCFile::Open` to open the rdc file
- `CaptureFile::OpenCapture` returns an `IReplayController`
  - `ReplayController::CreateDevice`
    - `Vulkan_CreateReplayDevice` creates a `WrappedVulkan`
      - the state is initially `CaptureState::LoadingReplaying`
    - `PostCreateInit`
      - `VulkanReplay::ReadLogInitialisation` replays the rdc file
        - `WrappedVulkan::ProcessChunk` replays a chunk
        - this replays until `SystemChunk::CaptureScope` chunk
          - which appears to replay resource creations
        - `ContextReplayLog` replays `SystemChunk::CaptureScope` chunk
          - which appears to replay `vkCmd*`
- os-specific `DisplayRendererPreview`
  - it creates an xcb window on linux
  - `ReplayController::CreateOutput` creates a `ReplayOutput` for the window
  - `ReplayController::SetFrameEvent` replays the specified event
    - `VulkanReplay::ReplayLog` with `eReplay_WithoutDraw`
    - `VulkanReplay::ReplayLog` with `eReplay_OnlyDraw`
  - `ReplayOutput::Display` displays the frame
    - `ReplayOutput::DisplayTex`
      - `ReplayOutput::ClearBackground` clears the background to checkboard
      - `VulkanReplay::RenderTexture` draws the frame to the window
    - `VulkanReplay::FlipOutputWindow` presents
