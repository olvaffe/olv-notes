Mesa on Android
===============

## VNDK

- Android framework provides some shared libraries for use by apps and hals
  - NDK is for apps
  - VNDK is for hals, which consists of
    - LLNDK, for hals and sp-hals and is a superset of NDK
    - VNDK-SP, for hals and sp-hals
    - VNDK, for hals
- vndk build system
  - <https://source.android.com/docs/core/architecture/vndk/build-system>
  - a `core` library resides in `/system` and is available only to `/system`
    - `cc_library` without other properties
  - a `vendor-only` library resides in `/vendor` and is available only to
    `/vendor`
    - `cc_library` with `proprietary` or `vendor`
  - a `vendor_available` library has two copies, one in `/system` and one in
    `/apex/com.android.vndk.v${VER}`
    - `cc_library` with `vendor_available` and no `vndk.enabled`
  - a `vndk` library resides in `/apex/com.android.vndk.v${VER}` and is
    available only to `/vendor`
    - `cc_library` with `vendor_available` and `vndk.enabled`
  - a `vndk-sp` library resides in `/apex/com.android.vndk.v${VER}` and is
    available only to `/vendor`
    - `cc_library` with `vendor_available`, `vndk.enabled`, and
      `vndk.support_system_process`
    - for use by `SP-HAL` hal, which can be loaded into a `/system` process
  - an `llndk` library resides in `/system` and is available to both `/system`
    and `/vendor`
    - `cc_library` with `llndk.symbol_file`
    - an `ndk` library of the same name exists, but with fewer public symbols
- interesting `llndk` libraries
  - <https://cs.android.com/search?q=%22llndk:%22%20file:Android.bp&sq=>
  - `libc`, `libm`, `libdl`
  - `libEGL`, `libGLESv1_CM`, `libGLESv2`, `libvulkan`
  - `libnativewindow`
    - `android/native_window.h`
    - `android/hardware_buffer.h`
    - `vndk/window.h`
    - `vndk/hardware_buffer.h`
  - `libsync`
    - `android/sync.h`
  - `liblog`
    - `android/log.h`
    - `log/log.h`
  - guessing from the symbol files,
    - a symbol without comment is available to apps and hals
    - a symbol with `# introduced=$VER` is avalable to apps and hals since
      `$VER`
    - a symbol with `# llndk` is available to hals but not apps
- interesting `vndk-sp` libraries
  - <https://cs.android.com/search?q=%22support_system_process:%22%20file:Android.bp&sq=>
    - simply <https://cs.android.com/android/platform/superproject/+/master:prebuilts/vndk/v33/x86_64/Android.bp> for v33
  - `libc++`, `libz`
  - `libcutils`
    - `cutils/properties.h`
    - `cutils/trace.h`
    - `cutils/native_handle.h`
  - `libutilscallstack`
    - `utils/CallStack.h`
  - `libunwindstack`
    - this obsoletes `libbacktrace` which has been removed
  - `libhardware`
    - `hardware/hardware.h`
    - `hardware/gralloc.h`
    - this has been deprecated
  - `android.hardware.graphics.common-V3-ndk`
  - `android.hardware.graphics.mapper@4.0`
  - `android.hardware.graphics.allocator-V1-ndk`
  - `android.hardware.graphics.composer3-V1-ndk`

## `android-stub`

- `android/` is a part of NDK
- `backtrace/` has been removed
- `cutils/` is a part of vndk-sp
- `hardware/` is a part of vndk-sp
- `log/` is a part of LLNDK
- `nativebase/` is a part of vndk-sp
- `vndk/` is a part of vndk-sp
