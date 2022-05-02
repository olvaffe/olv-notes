ANGLE
=====

## Build and Use

- <https://chromium.googlesource.com/angle/angle/+/HEAD/doc/DevSetup.md>
  - `git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git`
  - `export PATH=`pwd`/depot_tools:$PATH`
  - `git clone https://chromium.googlesource.com/angle/angle`
  - `cd angle`
  - `python scripts/bootstrap.py`
  - `gclient sync`
  - `sudo ./build/install-build-deps.sh`
  - `gn gen out/Debug`
  - `autoninja -C out/Debug`
- Run
  - export `LD_LIBRARY_PATH=<path-to-angle>/out/Debug`
  - `ldd es2gears`
  - `es2gears`
  - `ANGLE_DEFAULT_PLATFORM=vulkan es2gears`
    - this does not work because the native window is considered invalid
      - iirc, `xcb_create_window` != `XCreateWindow` and angle expects xcb
    - glmark2 with `-Dflavors=x11-glesv2` works
- Android
  - <https://chromium.googlesource.com/angle/angle/+/HEAD/doc/DevSetupAndroid.md>
  - `echo "target_os = ['android']" >> .gclient`
  - `gclient sync`
  - `gn args out/Android`
    - `target_os = "android"`
    - `target_cpu = "arm64"`
    - `is_component_build = false`
    - `is_debug = false`
    - `use_goma = true`
  - `autoninja -C out/Android`
  - `adb install -r -d --force-queryable out/Android/apks/AngleLibraries.apk`
    - verify installation with `adb shell pm path org.chromium.angle`
  - enable angle
    - `adb shell settings put global angle_debug_package org.chromium.angle`
    - `adb shell settings put global angle_gl_driver_all_angle 1`
  - disable
    - `adb shell settings delete global angle_gl_driver_all_angle`
    - `adb shell settings delete global angle_debug_package`
  - to verify, `adb logcat -d | grep ANGLE`
- cross compile
  - `./build/linux/sysroot_scripts/install-sysroot.py --arch=arm64`
  - `gn args out/AArch64`
    - `target_os = "linux"`
    - `target_cpu = "arm64"`
    - `is_component_build = false`
    - `is_debug = false`
    - `use_goma = true`
  - `autoninja -C out/AArch64`
  - package
    - `cd out`
    - `angledir=angle-aarch64-$(date +%Y%m%d)`
    - `ln -sf AArch64 $angledir`
    - `find -H $angledir -maxdepth 1 -type f -executable | xargs aarch64-linux-gnu-strip -x`
    - `find -H $angledir -maxdepth 1 -type f -executable | xargs tar zcf $angledir.tar.gz $angledir/gen`
    - after unpack,
      - `ln -sf libEGL.so libEGL.so.1`
      - `ln -sf libGLESv2.so libGLESv2.so.2`

## `eglGetDisplay`

- <https://chromium.googlesource.com/angle/angle/+/refs/heads/main/extensions/EGL_ANGLE_platform_angle.txt>
- stack
    #0 egl::Display::GetDisplayFromNativeDisplay
    #1 egl::GetDisplay
    #2 EGL_GetDisplay
    #3 eglGetDisplay
- `GetDisplayFromNativeDisplay`
  - platform type is `EGL_PLATFORM_ANGLE_ANGLE`
  - display type is `EGL_PLATFORM_ANGLE_TYPE_OPENGL_ANGLE`
    - when `ANGLE_DEFAULT_PLATFORM` is not set, it is mapped to
      `EGL_PLATFORM_ANGLE_TYPE_OPENGL_ANGLE`
    - `ANGLE_DEFAULT_PLATFORM=gl` maps it to
      `EGL_PLATFORM_ANGLE_TYPE_OPENGLES_ANGLE`
    - `vulkan` or `swiftshader` maps it to
      `EGL_PLATFORM_ANGLE_TYPE_VULKAN_ANGLE`
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_TYPE_ANGLE`
  - device type is `0`
    - it is mapped to `EGL_PLATFORM_ANGLE_DEVICE_TYPE_HARDWARE_ANGLE`
    - `ANGLE_DEFAULT_PLATFORM=swiftshader` maps it to
      `EGL_PLATFORM_ANGLE_DEVICE_TYPE_SWIFTSHADER_ANGLE`
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_DEVICE_TYPE_ANGLE`
  - platform type is `0`
    - it is mapped to `EGL_PLATFORM_X11_EXT` on Linux/X11
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_NATIVE_PLATFORM_TYPE_ANGLE`

## `eglInitialize`

- stack
    #0  rx::RendererVk::initialize
    #1  rx::DisplayVk::initialize
    #2  rx::DisplayVkXcb::initialize
    #3  egl::Display::initialize
    #4  egl::Initialize
    #5  EGL_Initialize
    #6  eglInitialize

## Android

- `GraphicsEnvironment.java`
  - `getDriverForPackage`
    - if `ANGLE_GL_DRIVER_ALL_ANGLE`, return `angle`
    - else check `ANGLE_GL_DRIVER_SELECTION_PKGS` and
      `ANGLE_GL_DRIVER_SELECTION_VALUES`
  - `shouldUseAngle` returns true when `getDriverForPackage` returns `angle`
  - `getAngleDebugPackage` returns `ANGLE_DEBUG_PACKAGE`
  - `setAngleInfo` passes angle info to native code
