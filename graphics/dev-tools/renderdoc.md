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

## `renderdoccmd`

- capture
  - `renderdoccmd capture -c <output-name> -w <executable> ...`
  - might want `-d` to specify the working dir
- replay
  - `renderdoccmd replay <capture.rdc>` just works
  - `renderdoccmd remoteserver` just works for remote replay

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

## `VK_LAYER_RENDERDOC_Capture`

- `library_loaded` is called upon `dlopen`
  - `RenderDoc::Initialise`
    - `Network::CreateServerSocket` listens on port
      `RenderDoc_FirstTargetControlPort`
  - `LibraryHooks::RegisterHooks` registers hooks for various apis
    - `EGLHook::RegisterHooks` early returns on android because of
      `EGL_ANDROID_GLES_layers`
    - `GLHook::RegisterHooks` early returns on android similarly
    - `VulkanHook::RegisterHooks` inits itself
- app calls `vkCreateInstance` and ends up in `hooked_vkCreateInstance`
  - `KeepLayerAlive` creates an (ununsed) internal instance on android
  - `WrappedVulkan::vkCreateInstance`
    - `RenderDoc::AddDeviceFrameCapturer` creates a capturer
