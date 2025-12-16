Mesa meson
==========

## Dependencies

- `python -m venv --system-site-packages ~/.pip`
  - `pip install packaging mako pyyaml`
- some drivers (e.g., intel, panfrost, asahi) and frontends (e.g., rusticl)
  compile clc to spirv, and they depend on
  - `libclang-cpp.so`, or its static equivalent, to compile clc to llvm ir
  - `libLLVM.so`, or its static equivalent, to process llvm ir
  - `LLVMSPIRVLib`, to translate llvm ir to spirv
  - `libclc`, to link spirv with `spirv64-mesa3d-.spv` (or `spirv-mesa3d-.spv`)
  - `SPIRV-Tools`, to link, parse, specialize, disassemble spirv
  - `pacman -S clang llvm spirv-llvm-translator libclc glslang`
- platform wayland depends on
  - `wayland-client`, to talk to server
  - `wayland-protocols`, for various protocols
  - `wayland-scanner`, to generate protocol code
  - `wayland-server`, for legacy `wl_drm`
  - `wayland-egl-backend`, for egl
- platform x11 depends on
  - `xcb`, for core
  - `xcb-xfixes`, for xfixes
  - `xcb-dri3`, for dri3
  - `xcb-present`, for present
  - `xcb-sync`, for sync (with xshmfence)
  - `xshmfence`, for xshmfence
  - `xcb-randr`, for randr (to detect xwayland, refresh rate, etc.)
  - `xcb-shm`, for swrast
  - `x11-xcb`, to fish `xcb_connection_t` out of `Display`
  - `xrandr`, not used but `VK_EXT_acquire_xlib_display` requires the header
  - glx supports indirect glx, and mixes xlib and xcb
    - `xcb-glx`, for indirect glx
    - `xcb-dri2`, for dri2
    - `glproto`, for indirect gl cmds
    - `dri2proto`, for xlib dri2
    - `xfixes`, for xlib xfixes
    - `xxf86vm`, to get current modeline for `GLX_OML_sync_control`
- misc dependencies
  - `zlib`, for disk cache
  - `libzstd`, for disk cache
  - `expat`, for drirc
  - `perfetto`, for perfetto
  - `libglvnd`, for glvnd
  - `libva`, for va
  - `libelf`, for amd llvm compiler
  - `libudev`, for `VK_EXT_display_control`

## Common Options

- vulkan-only build
  - `-Dgallium-drivers= -Dvulkan-drivers=foo`
  - all other apis depending on gallium are automatically disabled
- gallium-only build
  - `-Dgallium-drivers=foo -Dvulkan-drivers=`
  - if the system is GLVND-enabled, `-Dglvnd=enabled`
  - OpenGL/ES are enabled automatically
    - to disable OpenGL, `-Dopengl=false`
    - to disable ES, `-Dgles1=false` and `-Dgles2=false`
  - EGL/GLX/GBM are enabled automatically when OpenGL/ES are enabled
    - to disable EGL, `-Degl=disabled`
    - to disable GLX, `-Dglx=disabled`
    - to disable GBM, `-Dgbm=disabled`
  - VAAPI driver is enabled automatically,
    - to disable VAAPI driver, `-Dgallium-va=disabled`
    - to enable proprietary codecs, `-Dvideo-codecs=all`
- one-liners
  - radeonsi-only build
    - `meson setup out-radeonsi -Dbuildtype=debug -Dgallium-va=disabled -Dgallium-drivers=radeonsi -Dvulkan-drivers=`
  - vaapi-only build
    - `meson setup out-vaapi -Dbuildtype=debug -Dplatforms= -Dglx=disabled -Degl=disabled -Dgbm=disabled -Dopengl=false -Dgles1=disabled -Dgles2=disabled -Dvideo-codecs=all -Dgallium-drivers=radeonsi -Dvulkan-drivers=`
  - rusticl-only build
    - `meson setup out-rusticl -Dbuildtype=debug -Dplatforms= -Dglx=disabled -Degl=disabled -Dgbm=disabled -Dopengl=false -Dgles1=disabled -Dgles2=disabled -Dgallium-va=disabled -Dgallium-rusticl=true -Dgallium-drivers=radeonsi -Dvulkan-drivers=`
    - `pacman -S rust rust-bindgen libclc spirv-llvm-translator llvm clang`

## Options

- `allow-fallback-for` allows specified deps (e.g., `libdrm`, `libva`, and `perfetto`)
  to use wraps
- `allow-kcmp` defines `-DALLOW_KCMP`
  - this is desirable because of seccomp
  - this provides `os_same_file_description` impl
- `amdgpu-virtio` enables virtio support for radeonsi and radv
- `amd-use-llvm` enables llvm support, in addition to aco, for radeonsi and
  radv
- `android-libbacktrace` defines `-DWITH_LIBBACKTRACE`
  - it also builds `backtrace` stub
  - this provides `u_debug_stack.h` impl on android
- `android-libperfetto` defines `ANDROID_LIBPERFETTO` to use android's system
  perfetto rather than sdk
- `android-strict` defines `-DANDROID_STRICT`
  - this caps vk versions/extensions for android
- `android-stub` builds various stub shared libraries for android
  - this enables an android build without android deps
- `build-aco-tests` builds `aco_tests` unit tests
- `build-radv-tests` builds `radv_tests` unit tests
- `build-tests` builds various unit tests
- `custom-shader-replacement` defines `-DCUSTOM_SHADER_REPLACEMENT` for custom
  shader replacement for mesa main
  - the built-in shader replacement based on `MESA_SHADER_DUMP_PATH` and
    `MESA_SHADER_READ_PATH` should be good enough
- `d3d-drivers-path` is the install path of `d3dadapter9.so`
  - default to `$libdir/d3d`
  - it is a driver loaded by wine nine
- `datasources` enables various perfetto datasources
  - they are supported drivers of `pps-producer`
- `display-info` uses `libdisplay-info` for `VK_KHR_display` hdr
- `draw-use-llvm` enables llvm for draw module (all stages before
  rastreization), required by llvmpipe
- `dri-drivers-path` is the install path of `*_dri.so`
  - default to `$libdir/dri`
  - nowadays, egl/glx/gbm directly links to `libgallium.so`, which provides
    the real drivers
  - `*_dri.so` is for legacy X11
- `egl` builds EGL
- `egl-lib-suffix` specifies the suffix for `libEGL.so`, for android
- `egl-native-platform` defines `_EGL_NATIVE_PLATFORM`
  - it is the default platform when none is specified and EGL cannot guess it
- `enable-glcpp-tests` enables `glcpp` unit tests
- `execmem` has no effect
  - it was used to generate dynamic stubs for glapi
- `expat` is an internal detail
  - `xmlconfig` and intel tools both require `expat`
- `freedreno-kmds` specifies supported freedreno kmds
  - there are `msm`, `kgsl`, `virtio`, and `wsl`
- `gallium-d3d10-dll-name` is the name of d3d10 umd dll
  - default to `libgallium_d3d10.dll`
- `gallium-d3d10umd` builds sw d3d10 umd for windows
  - it uses llvmpipe/softpipe
- `gallium-d3d12-graphics` enables gfx support in d3d12 gallium driver
- `gallium-d3d12-video` enables video support in d3d12 gallium driver
- `gallium-drivers` specifies the list of enabled gallium drivers
  - hw drivers
  - `zink` and `d3d12` are layered drivers on top of vk/d3d12
  - `svga` and `virgl` are virtualized drivers for vmware/virtio-gpu
  - `llvmpipe` and `software` are sw drivers
  - `kmsro` is for when the drm fd lacks a rendernode
- `gallium-extra-hud` defines `-DHAVE_GALLIUM_EXTRA_HUD=1`
  - it provides `hud_create` used by gallium fronts to provide HUD info
- `gallium-mediafoundation` enables gallium mediafoundation frontend
- `gallium-rusticl` enables gallium opencl frontend/target
  - similar to vk, this builds an opencl icd loaded by `OpenCL-ICD-Loader`
- `gallium-rusticl-enable-drivers` lists enabled drivers
  - most drivers require setting `RUSTICL_ENABLE` at runtime
- `gallium-va` enables gallium vaapi frontend/target
- `gallium-wgl-dll-name` is the name of wgl dll
  - default to `libgallium_wgl.dll`
  - it provides sw gl on windows
- `gbm` builds GBM
- `gbm-backends-path` defines `-DDEFAULT_BACKENDS_PATH=`
  - defaults to `$libdir/gbm`
  - it is the search path for (external) gbm backends
- `gles1` defines `-DHAVE_OPENGL_ES_1=1`
  - it enables gles1 support in egl and mesa main
- `gles2` defines `-DHAVE_OPENGL_ES_2=1`
  - it enables gles2 support in egl and mesa main
- `gles-lib-suffix` specifies the suffix for `libGLESv1_CM.so` and
  `libGLESv2.so`, for android
- `glvnd` defines `-DUSE_LIBGLVND=1`
  - for glx, it defines `__glx_Main` and renames `libGL.so.1.2.0` to
    `libGLX_mesa.0.0.0`
  - for egl, it defines `__egl_Main` and renames `libEGL.so.1.0.0` to
    `libEGL_mesa.0.0.0`
- `glvnd-vendor-name` specifies the glvnd vendor name
  - default to `mesa`
- `glx` builds GLX
  - with `xlib`, it is pure sw and uses only core x11 protocol
  - with `dri`, it always defines `-DGLX_INDIRECT_RENDERING` and uses glx
    protocol for indirect rendering
- `glx-direct` defines `-DGLX_DIRECT_RENDERING`
  - it uses various protocols for direct rendering
- `glx-read-only-text` defines `-DGLX_X86_READONLY_TEXT`
  - it affects glapi dispatch
- `gpuvis` defines `-DHAVE_GPUVIS`
  - it enables gpuvis tracing
- `html-docs` builds docs
- `html-docs-path` is the install path of docs
  - default to `$datadir/doc/mesa`
- `imagination-srv` defines `-DPVR_SUPPORT_SERVICES_DRIVER`
  - it enables downstream `pvr` kmd support
- `imagination-uscgen-devices`
- `install-intel-gpu-tests` installs `intel_FOO_mi_builder_test` unit tests
- `install-mesa-clc` installs `mesa_clc`
- `install-precomp-compiler` installs `asahi_clc` and `panfrost_compile`
- `intel-elk` enables intel gen8- compiler support
- `intel-rt` enables raytracing in intel anv driver
- `legacy-wayland` enables `EGL_WL_bind_wayland_display` support for egl
- `libgbm-external` links against system libgbm
- `libunwind` defines `-DHAVE_LIBUNWIND`
  - this provides `u_debug_stack.h` impl on linux
- `llvm` explicitly enables/disables llvm support
  - llvm is required by llvmpipe, rusticl, clc, etc.
- `llvm-orcjit` uses ORCJIT instead of MCJIT for llvmpipe
- `lmsensors` enables sensors support for gallium hud
- `mediafoundation-codecs` selects enabled codecs for gallium mediafoundation
- `mesa-clc` builds `mesa_clc` and `vtn_bindgen2`
  - `mesa_clc` translates CLC to SPIRV
  - `vtn_bindgen2` translates SPIRV to NIR, and generates a C function to
    build the NIR
  - `system` uses pre-built `mesa_clc` and `vtn_bindgen2`
- `mesa-clc-bundle-headers` bundles `opencl-c.h` into `mesa_clc`
- `microsoft-clc` builds `clon12compiler.dll`
  - it translates CL to SPIRV to NIR to DXIL
- `min-windows-version` specifies win ver
  - it defines `-DWINDOWS_NO_FUTEX` on win 8 and before
- `moltenvk-dir` defines the path to moltenvk
  - it is used by zink
- `opengl` defines `-DHAVE_OPENGL=1`
  - it enables gl support in egl and mesa main
- `perfetto` defines `-DHAVE_PERFETTO`
  - it enables perfetto tracing and `pps-producer`
- `platforms` specifies supported window systems
  - user can specify `x11`, `wayland`, `haiku`, `android`, `windows`, `macos`
  - `xcb` is automatically added when `x11` is selected
  - egl adds `surfaceless`, and if gbm is enabled, `drm`
- `platform-sdk-version` specifies the android sdk version, defaults to 34
- `power8` enables power8 optimizations
- `precomp-compiler` builds `asahi_clc` and `panfrost_compile`
  - they translate SPIRV to NIR to hw-specific binary, and generate a C
    variable to embed the binary
- `radeonsi-build-id` defines `-DRADEONSI_BUILD_ID_OVERRIDE=`
- `radv-build-id` defines `-DRADV_BUILD_ID_OVERRIDE=`
  - it is used to avoid shader cache rebuild when two radv versions are known
    to generate compat shader binaries
- `selinux` has no effect
  - it was used with `execmem`
- `shader-cache` defines `-DENABLE_SHADER_CACHE`
  - it enables mesa disk cache
- `shader-cache-default` defines `-DSHADER_CACHE_DISABLE_BY_DEFAULT` if
  disabled
  - it disables disk cache by default and requires
    `MESA_SHADER_CACHE_DISABLE=1` envvar to re-enable
- `shader-cache-max-size` defines `-DMESA_SHADER_CACHE_MAX_SIZE=`
  - default to 1GB or `MESA_SHADER_CACHE_MAX_SIZE` envvar
- `shared-glapi` builds `libglapi.so`
  - it provides current ctx and dispatch table shared by gl, gles1, and gles2
  - it might not be needed with `glvnd`?
  - deprecated and always enabled
- `shared-llvm` defines `-DLLVM_IS_SHARED=`
  - it links llvm statically/dynamically
- `spirv-to-dxil` enables spirv to dxil translation
  - this is needed by vk-to-d3d12
  - it also provides offline `spirv_to_dxil` compiler
- `spirv-tools` defines `-DHAVE_SPIRV_TOOLS`
- `split-debug` adds `-gsplit-dwarf` and `-Wl,--gdb-index` for a debug build
  - `-gsplit-dwarf` splits dwarf info to a separate `.dwo` file
  - `--gdb-index` adds `.gdb-index` elf section to speed up gdb
- `sse2` enables sse2 optimization
- `static-libclc` embeds libclc spirv statically
  - otherwise, it is read dynamically from
    `/usr/lib/clc/{spirv,spirv64}-mesa3d-.spv`
- `sysprof` enables sysprof-backend for utrace
- `teflon` enables gallium tensorflow lite frontend/target
- `tools` enables various tools
  - generic: `drm-shim`, `glsl`, `nir`, `dlclose-skip`
  - driver-specific: `etnaviv`, `freedreno`, `intel`, `intel-ui`, `nouveau`,
    `lima`, `panfrost`, `asahi`, `imagination`,
- `unversion-libgallium` builds `libgallium_dri.so` instead of
  `libgallium-<version>.so`
- `valgrind` defines `-DHAVE_VALGRIND`
  - it helps valgrind does its job
- `va-libs-path` specifies the install path for vaapi drivers
  - default to `$libdir/dri`
- `video-codecs` defines `-DVIDEO_CODEC_FOO=1`
  - `{vc1,h264,h265,av1,vp9}{dec,enc}`, or simply `all` or `all_free`
  - it affects anv, radv, gallium-on-d3d12, and `vl_codec_supported` used by
    vaapi
- `virtgpu_kumquat` enables kumquat ffi for gfxstream
- `vmware-mks-stats` defines `-DVMX86_STATS=1` for vmware
- `vulkan-beta` defines `-DVK_ENABLE_BETA_EXTENSIONS`
  - it enables beta extensions
- `vulkan-drivers`
  - hw drivers
  - `virtio` and `gfxstream` are virtualized drivers for virtio-gpu
  - `swrast` is sw driver using llvmpipe
  - `microsoft-experimental` is layered driver on top of d3d12
- `vulkan-icd-dir` specifies the install path for vk icd jsons
  - default to `$datadir/vulkan/icd.d`
- `vulkan-layers` specifies vk layers to build
  - `anti-lag` supports `VK_AMD_anti_lag`
  - `device-select` makes the best device the first
  - `intel-nullhw` disables draw and compute at hw level
  - `overlay` draws hud using imgui
  - `screenshot` writes presented images to png
    `vram-report-limit` caps reported mem heap sizes and budgets
- `vulkan-manifest-per-architecture` decides if `<arch>` part is included in
  the generated `<drv>_icd.<arch>.json`
- `xlib-lease` enables `VK_EXT_acquire_xlib_display` support
- `xmlconfig` defines `-DWITH_XMLCONFIG=`
  - it enables drirc support
  - when disabled, `driconf_static.h` is generated at compile time
- `zlib` defines `-DHAVE_ZLIB` and `-DHAVE_COMPRESSION`
  - it is used by disk cache and crc32
- `zstd` defines `-DHAVE_ZSTD` and `-DHAVE_COMPRESSION`
  - it is used by disk cache

## Android

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
