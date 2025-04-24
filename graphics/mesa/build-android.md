Mesa on Android
===============

## Build

- <https://docs.mesa3d.org/android.html>
- compile mesa-clc (if panvk)
  - `apt install llvm llvm-dev clang libclang-dev libclang-cpp-dev llvm-spirv-19 libclc-19-dev glslang-tools`
  - `meson setup out-host -Dprefix=/tmp/mesa -Dbuildtype=release -Dgallium-drivers= -Dvulkan-drivers=panfrost -Dplatforms= -Dinstall-mesa-clc=true -Dinstall-precomp-compiler=true`
- cross-compile drm
  - `meson setup --cross-file ndk.ini out-ndk -Dprefix=/tmp/mesa -Dbuildtype=release`
- cross-compile llvm (if llvmpipe)
  - `cmake -Sllvm -Bout-ndk -GNinja -DCMAKE_INSTALL_PREFIX=/tmp/mesa -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `-DCMAKE_TOOLCHAIN_FILE=~/android/sdk/ndk/28.0.13004108/build/cmake/android.toolchain.cmake -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-34`
  - `-DLLVM_TARGETS_TO_BUILD=AArch64 -DLLVM_INCLUDE_TOOLS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF`
  - android x86
    - `-DANDROID_ABI=x86_64 -DLLVM_TARGETS_TO_BUILD=X86` instead
- compile llvm-config (if llvmpipe)
  - `cmake -Sllvm -Bout-host -GNinja -DCMAKE_INSTALL_PREFIX=/tmp/mesa -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `-DLLVM_TARGETS_TO_BUILD=AArch64 -DLLVM_BUILD_TOOLS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_ENABLE_ZSTD=OFF -DLLVM_ENABLE_LIBXML2=OFF -DHAVE_LIBRT=OFF`
  - `ninja -C out-host llvm-config`
  - `cp out-host/bin/llvm-config /tmp/mesa/bin`
- common mesa options
  - `-Dplatforms=android -Dplatform-sdk-version=34 -Dandroid-strict=true -Dandroid-stub=true -Dandroid-libbacktrace=false -Degl-lib-suffix=_mesa -Dgles-lib-suffix=_mesa -Dglx=disabled -Dgbm=disabled`
- cross-compile panvk
  - `meson setup --cross-file ndk.ini out-ndk -Dprefix=/tmp/mesa -Dbuildtype=debug -Dgallium-drivers=panfrost -Dvulkan-drivers=panfrost -Dmesa-clc=system -Dprecomp-compiler=system`
  - plus common mesa options
- cross-compile llvmpipe
  - `meson setup --cross-file ndk.ini out-ndk -Dprefix=/tmp/mesa -Dbuildtype=debug -Dgallium-drivers=llvmpipe -Dvulkan-drivers=swrast`
  - `-Dcpp_rtti=false -Dshared-llvm=disabled`
  - plus common mesa options
  - if build error, patch `meson.build` to add `selectiondag` to `llvm_modules`

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
