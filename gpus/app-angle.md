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
    - glmark2 with `-Dflavors=x11-glesv2` works

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
