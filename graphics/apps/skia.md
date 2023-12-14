Skia
====

## Build

- <https://skia.org/docs/user/build/>
  - `git clone https://skia.googlesource.com/skia.git`
  - `cd skia`
  - `./tools/git-sync-deps`
  - `./tools/install_dependencies.sh`
  - `./bin/fetch-gn`
  - `./bin/gn gen out`
  - `ninja -C out`
- update
  - `git pull`
  - `./tools/git-sync-deps`
- args
  - `gn args out --list` to see the current values
    - `is_component_build = false`
      - true for shared library
      - false for static library
    - `is_debug = true`
      - true for no optimization
      - false for `-O3` and `-DNDEBUG`
    - `is_official_build = false`
      - true to disable `is_debug`
      - false to enable `is_debug` and to add `-g`
    - `skia_enable_ganesh = true`
      - the old gpu backend
    - `skia_enable_graphite = false`
      - the new gpu backend
    - `skia_enable_gpu_debug_layers = true`
      - enable vk validation layer for tests/tools
    - `skia_enable_spirv_validation = true`
      - enable vk validation layer for tests/tools
    - `skia_gl_standard = ""`
      - defines `SK_ASSUME_GL_ES=1` or `SK_ASSUME_GL=1` to reduce code size
    - `skia_use_angle = false`
    - `skia_use_egl = false`
      - true to use egl rather than glx
      - always true on android
    - `skia_use_gl = true`
      - true to enable gl/gles backend
    - `skia_use_vulkan = false`
      - true to enable vk backend
    - `skia_use_x11 = true`
      - use glx unless `skia_use_egl`
  - disable font
    - `skia_enable_fontmgr_android = false`
    - `skia_enable_fontmgr_empty = true`
  - disable optional features
    - `skia_enable_pdf = false`
  - cross-compile
    - `target_os = "linux"`
    - `target_cpu = "arm64"`
    - `target_cc = "/usr/bin/aarch64-linux-gnu-gcc"`
    - `target_cxx = "/usr/bin/aarch64-linux-gnu-g++"`
    - `extra_cflags = [ "--sysroot=/sysroot-arm64", "-Wno-error" ]`
    - `extra_ldflags = [ "--sysroot=/sysroot-arm64" ]`
  - misc
    - `cc_wrapper = "ccache"`
- linux build
  - args
    - `is_component_build = true`
    - `skia_use_egl = true`
    - `skia_use_vulkan = true`
    - `cc_wrapper = "ccache"`
  - meson
    - use `cpp.find_library('skia')` directly
    - comparing defines in `defines.bzl` and `include`, we should define
      - `SK_DEBUG`
      - `SK_GANESH`
      - `SK_GL`
      - `SK_VULKAN`
  - deploy
    - `strip -g out/lib*.so`
    - `tar -zcf skia-dist.tar.gz --transform="s,,skia-dist/," out/lib*.so include src/base/SkTime.h modules/skcms/skcms.h modules/skcms/src/skcms_public.h`

## Tests

- terms
  - gm stands for golden master test suite
  - dm stands for dungeon master and runs unit tests and gms
    - first commit says "dm is like gm, but faster and with fewer features"
  - fm was an attemp to become better dm?
    - first commit says "FM, a dumber new testing tool"
- <https://skia.org/docs/dev/testing/>
  - `./out/fm --resourcePath resources --sources arithmode --backend vk`
    - `--listTests`
    - `--listGMs`
  - `./out/dm --resourcePath resources --src gm --config vk --verbose`
  - see below for skqp
- tests
  - `DEF_GANESH_TEST_*`
    - defines a test of type `skiatest::TestType::kGanesh`
    - a gpu unit test
    - on older branches, this is `DEF_GPUTEST`
  - `DEF_TEST`
    - defines a test of type `skiatest::TestType::kCPU`
    - a cpu unit test
  - `DEF_GM`
    - defines a GM test

## SkQP

- tot is from <https://skia.googlesource.com/skia.git>
- cts branches are from <https://android.googlesource.com/platform/external/skqp>
- tests
  - on tot, skqp only runs gpu (ganesh) unit tests
    - see `get_unit_tests`
  - on `android11-tests-release`, it runs
    - gm tests (`gles_`, `vk_`, and `gl_`)
      - see `get_render_tests`
    - gpu unit tests (`unitTest_`)
      - see `get_unit_tests`
  - on `pie-cts-release`, it runs
    - gm tests (`SkiaGM_gles.*`, `SkiaGM_vk.*`, and `SkiaGM_gl.*`)
    - all unit tests (`Skia_Unit_Tests.*`)
    - see `register_skia_tests`
- linux build on tot
  - edit `tools/skqp/src/skqp.h` to set `fEnforcedAndroidAPILevel` to 99
    - otherwise, all tests are skipped
  - `./out/skqp . report` to run all tests
    - `.` is the asset dir: skqp looks for `resources` under the asset dir
    - `report` is for test results
- linux build on `android11-tests-release`
  - install `python-is-python2`
  - edit `DEPS` and `BUILD.gn` to remove all mentions of `Nima`
  - `tools/git-sync-deps`
  - use args from `tools/skqp/skqp_gn_args.py`, `tools/skqp/generate_gn_args`,
    and `tools/skqp/gn_to_bp.py`, and add
    - `skia_use_x11 = false`
    - `skia_use_egl = true`
    - `extra_cflags = [ "-Wno-error" ]`
  - fix trivial compile errors
    - remove duplicated definitions from `include/config/SkUserConfig.h`
    - add `#include <limits>` to
      `third_party/externals/spirv-tools/source/validate_cfg.cpp`
    - fix aarch64-specific issues by using the slow paths (or use clang)
      - `src/opts/SkRasterPipeline_opts.h`
  - `ninja skqp`
    - note that many tests do not build in this config because we disable pdf
      and skottie
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets report '^vk_'`
- linux build on `pie-cts-release`
  - similiar to on `android11-tests-release`
  - install `python-is-python2`
  - `tools/git-sync-deps`
  - use args from `tools/skqp/generate_gn_args` and `tools/skqp/gn_to_bp.py`,
    and add
    - `skia_use_egl = true`
    - `extra_cflags = [ "-Wno-error" ]`
  - fix trivial compile errors
    - remove duplicated definitions from `include/config/SkUserConfig.h`
    - fix aarch64-specific issues by using the slow paths (or use clang)
      - `src/jumper/SkJumper_stages.cpp`
      - `third_party/externals/sdl/src/atomic/SDL_spinlock.c`
      - `third_party/externals/skcms/src/Transform.c`
  - `ninja skqp`
    - note that many tests do not build in this config because we disable pdf
      and skottie
  - `./out/skqp --gtest_filter="SkiaGM_vk.*" platform_tools/android/apps/skqp/src/main/assets report`
- distribute
  - `mkdir skqp-dist`
  - `aarch64-linux-gnu-strip -xo skqp-dist/skqp out/skqp`
  - `ln -sf ../platform_tools/android/apps/skqp/src/main/assets skqp-dist/`
  - `tar zchf skqp-dist.tar.gz skqp-dist`
  - `rm -rf skqp-dist`
- old
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets skqp/rendertests.txt results`
    - `./out/skqp NOT USED results '^vk_arithmode$'` seems better

## Android

- skia on android
  - <https://android.googlesource.com/platform/external/skia> for platform
  - <https://android.googlesource.com/platform/external/skqp> for cts
  - <https://skia.org/docs/user/build/#android>
  - <https://skia.googlesource.com/skia/+/main/tools/skqp/README.md>
- to build an apk from skqp's `android12-tests-release` or
  `android11-tests-release` branch
  - install python2 and openjdk 8
    - `python-is-python2`
    - `openjdk-8-jre` and `openjdk-8-jdk`
  - install `ndk;16.1.4479499` and `platforms;android-26`
  - edit `DEPS` and `BUILD.gn` to remove all mentions of `Nima`
  - `tools/git-sync-deps`
  - `rm -rf platform_tools/android/apps/arcore`
  - comment out duplicated defines from `include/config/SkUserConfig.h`
  - `ANDROID_HOME=~/android/sdk ANDROID_NDK=~/android/sdk/ndk/16.1.4479499
    tools/skqp/make_universal_apk arm64`
- to build an apk from skqp's `pie-cts-release` branch
  - same as above
  - no need to remove Nima-Cpp and arcore
- note that `make_universal_apk` downloads models and sets up `resources`
  symlink
  - `tools/skqp/download_model`
  - `resources` symlink
- manual run
  - `adb install -r out/skqp/skqp-universal-debug.apk`
  - `adb shell am instrument -w org.skia.skqp`
  - `adb logcat org.skia.skqp skia "*:S"`
- CTS run
  - `./cts-tradefed run commandAndExit cts-dev -m CtsSkQPTestCases
       --module-arg 'CtsSkQPTestCases:include-filter:org.skia.skqp.SkQPRunner#gles_imageblur*'`

## `sk_app`

- `main`
  - it calls `sk_app::Application::Create` first, which is expected to
    - optionally call `SkGraphics::Init` to init skia
    - call `Window::CreateNativeWindow` to create a (hidden) native window
    - call `Window::pushLayer` to register for callbacks
    - call `Window::attach` to create a `SkSurface`
    - return an `Application`
  - it enters the main loop to dispatch events
    - if the window is resized, `Layer::onResize`
    - if the window is dirtied, `Layer::onPaint`
    - if there is mouse event, `Layer::onMouse` or `Layer::onMouseWheel`
    - if there is keyboard event, `Layer::onChar` or `Layer::onKey`
    - if there is touch event, `Layer::onTouch`, `Layer::onFling`, or
      `Layer::onPinch`
    - if there is no event, `Layer::onIdle`
  - it exits the main loop and calls `Application::~Application`, which is
    expected to
    - call `Window::detach` to destroy the `SkSurface`
    - delete the native window
- every time `Winwdow::attach` is called, `Layer::onBackendCreated` is called
  - app is expected to show the window and mark the window dirty
- every time repaint is needed, `Layer::onPaint` is called
  - app is expectd to repaint
  - `SkSurface::getCanvas` to get a `SkCanvas`
  - `SkCanvas::clear` to clear the surface
  - `SkCanvas::drawRect` to draw to the surface
  - `SkCanvas::drawSimpleText` to draw a string to the surface
- internally, if sw raster,
  - `Window::attach` creates a `SkSurface` before `Layer::onBackendCreated`
    - `SkImageInfo::Make(w, h, color, alpha)` to initialize a `SkImageInfo`
    - `SkSurface::MakeRaster` to create a sw `SkSurface`
  - after `Layer::onPaint`,
    - `SkSurface::flushAndSubmit` flushes and submits the gpu job (nop because
      sw raster)
    - `SkSurface::peekPixels` reads back the pixel data as a `SkBitmap`
    - sends the pixel data to the window system
- internally, if gl,
  - `Window::attach` creates a `GrGLInterface` and a `GrDirectContext` before
    `Layer::onBackendCreated`
    - it creates and makes a GL context current (this happens outside of skia)
    - `GrGLMakeNativeInterface` allocs a `GrGLInterface` using the current GL
      context
    - `GrDirectContext::MakeGL` creates a `GrDirectContext` using the
      `GrGLInterface`
      - this queries gpu caps, creates caches, etc.
  - before `Layer::onPaint`, a `SkSurface` is created on demand
    - it is cached until resize, etc.
    - a `GrBackendRenderTarget` is initialized
    - `SkSurface::MakeFromBackendRenderTarget` creates the `SkSurface`
      - the surface wraps a `skgpu::ganesh::Device`
      - canvas draw calls call into the device to record the command
  - after `Layer::onPaint`,
    - `SkSurface::flushAndSubmit` flushes and submits the gpu job
      - `GrDirectContext::flush`
      - `GrDirectContext::submit`
    - `glXSwapBuffers`
- internally, if vk,
  - `Window::attach` creates a `GrVkBackendContext`, a `GrDirectContext`, a
    `VkSwapchainKHR`, and multiple `SkSurface` before
    `Layer::onBackendCreated`
    - it initializes vk all the way outside of skia until it has a `VkQueue`
    - it fills in `GrVkBackendContext`
    - `GrDirectContext::MakeVulkan` creates a `GrDirectContext` using the
      `GrVkBackendContext`
      - this queries gpu caps, creates caches, etc.
    - a swapchain is created and `VkImage`s are quried back
    - `SkSurface::MakeFromBackendTexture` or
      `SkSurface::MakeFromBackendRenderTarget` are called to wrap `VkImage` in
      `SkSurface`
  - before `Layer::onPaint`, `vkAcquireNextImageKHR` is called to decide the
    next `VkSurface`
    - the aqcuire returns a in-`VkSemaphore`
    - `GrBackendSemaphore::initVulkan` and `SkSurface::wait` passes the
      in-semaphore to the surface
  - after `Layer::onPaint`,
    - `SkSurface::flushAndSubmit` flushes and submits the gpu job
      - `GrDirectContext::flush`
      - `GrDirectContext::submit`
    - it calls `SkSurface::flush` again and requests to signal an
      out-semaphore
    - it calls `GrDirectContext::submit` to really submit
    - it calls `vkQueuePresentKHR` with the out-sempahore

## Basics

- `SkPixmap` is a wrapper to cpu-access pixels
  - a pixmap is created from a pointer to pixels and a `SkImageInfo` to
    describe the pixels
  - skia cannot sample from or draw into a pixmap directly
  - skia can read from or write to a pixmap
    - while `SkCanvas` cannot sample from or draw into a pixmap, it can read
      the surface contents back to a `SkPixmap` prepared by the client
    - `SkPngEncoder` can encode a pixmap to a png
- `SkBitmap` owns a cpu memory for pixels
  - `pixelRef()` returns the cpu memory
    - the cpu memory is ref-counted and can be shared by bitmaps
  - `pixmap()` returns the pixmap (which is a wrapper to the storage)
  - a bitmap is as limited as a pixmap
- `SkImage` holds ro pixel data in some storage
  - skia can sample from an image directly
- `SkSurface` is the drawing destination
  - `SkSurfaces::Raster` creates a raster (sw) surface
    - internally, it uses a bitmap as the storage
  - `SkSurfaces::RenderTarget` creates a ganesh render target surface
- `SkCanvas` is the drawing context to a `SkSurface`
- `SkPicture` holds recorded `SkCanvas` drawing commands for playback later
- `SkImageInfo` describes a 2D RGBA image
  - `SkISize` for the width/height
  - `SkColorInfo`
    - `SkColorSpace` for the color space (srgb, p3, etc.)
    - `SkColorType` for pixel encoding (rgba 8888, etc.)
    - `SkAlphaType` for alpha interpretation (pre-multiplied, etc.)
- `SkYUVAInfo` describes a 2D YUVA image
  - `SkISize` for the width/height
  - `PlaneConfig` for planes (packed, bi-planar, tri-planar, etc.)
  - `Subsampling` for the subsampling (422, 420, etc.)
  - `SkYUVColorSpace` for the color space (rec 601, 709, etc.)
  - `SkEncodedOrigin` for the origin (top-left, etc.)
  - `Siting` for chroma siting (centered)
- `SkColorType`
  - `SkRasterPipeline::appendLoad` picks different opcodes depending on the
    color type
    - the opcodes are implemented in `SkRasterPipeline_opts.h`
  - `kRGB_565_SkColorType` uses `Op::load_565`
    - it is `VK_FORMAT_R5G6B5_UNORM_PACK16`
  - `kARGB_4444_SkColorType` uses `Op::load_4444`
    - it is `VK_FORMAT_R4G4B4A4_UNORM_PACK16`
  - `kRGBA_8888_SkColorType` uses `Op::load_8888`
    - it is `VK_FORMAT_A8B8G8R8_UNORM_PACK32`
  - `kRGBA_1010102_SkColorType` uses `Op::load_1010102`
    - it is `VK_FORMAT_A2B10G10R10_UNORM_PACK32`
  - `kRGB_888x_SkColorType` uses `Op::load_8888` and `Op::force_opaque`
  - `kBGRA_8888_SkColorType` uses `Op::load_8888` and `Op::swap_rb`

## GPU

- ganesh is the current gpu backend
  - graphite is the in-dev new gpu backend
- `GrDirectContext`
  - `GrDirectContext::MakeGL` creates GL-based direct context
    - clients can optionally provide `GrGLInterface` to choose an alternative
      GL driver
  - `GrDirectContext::MakeVulkan` creates a VK-based direct context
    - clients must provide `GrVkBackendContext` which specifies the `VkQueue`
      and others
- `SkSurface`
  - `SkSurface::MakeFromBackendTexture` creates a surface from a
    `GrBackendTexture`
  - `SkSurface::MakeFromBackendRenderTarget` creates a surface from a
    `GrBackendRenderTarget`
  - `SkSurface::MakeRenderTarget` creates a surface by allocating a rt
- `SkImage`
  - `BorrowTextureFrom` creates an image from a `GrBackendTexture`
  - `TextureFromYUVATextures` creates an image from a `GrYUVABackendTextures`
  - `PromiseTextureFrom` creates a promise image from `GrBackendFormat`
    - a promise image is an image whose backend texture is requested on-demand
    - it is useful because, when drawing to a `SkDeferredDisplayListRecorder`,
      no backend texture is requested
  - `PromiseTextureFromYUVA` creates a promise image from
    `GrYUVABackendTextureInfo`
- `GrBackendTexture` describes a backend (gl, vk, etc.) texture
  - `GrGLTextureInfo` consists of GL target, id, and format
  - `GrVkImageInfo` consists of
    - `VkImage`
    - `GrVkYcbcrConversionInfo`
    - vk image tiling, layout, format, usage, etc.
- `GrBackendRenderTarget` describes a backend rt
  - `GrGLFramebufferInfo` consists of GL fbo id and format
  - `GrVkBackendSurfaceInfo` consists of `GrVkImageInfo`
- `GrBackendFormat` describes a backend format
  - `GrBackendFormat:MakeGL` initializes from GL format and target
  - `GrBackendFormat:MakeVk` initializes from `VkFormat` or
    `GrVkYcbcrConversionInfo`

## Ganesh `GrVkCaps`

- `GrVkCaps::GrVkCaps` inits statically-determinted caps
- `GrVkCaps::init` inits dynamically-determinted caps
  - `GrVkCaps::initGrCaps`
  - `GrVkCaps::initShaderCaps`
  - `GrVkCaps::initFormatTable` inits `fFormatTable` and
    `fColorTypeToFormatTable`
  - `GrVkCaps::initStencilFormat` inits `fPreferredStencilFormat`
  - `GrVkCaps::applyDriverCorrectnessWorkarounds`
    - it looks like `fMustUseCoherentHostVisibleMemory` can hurt MTL
  - `GrCaps::finishInitialization`
    - `GrCaps::applyOptionsOverrides`
    - `SkCapabilities::initSkCaps`
- format tables
  - `fColorTypeToFormatTable` is initialized by `initFormatTable` and
    `setColorType`
    - skia core works with color types and the table is used to map color
      types to vk formats
  - `fFormatTable` is initialized by `initFormatTable` and `FormatInfo::init`
    - `fFormatTable` is an array of `FormatInfo`
    - `getFormatInfo` maps a `VkFormat` to a `FormatInfo` in `fFormatTable`
      using `kVkFormats`
    - `FormatInfo::init` queries `VkFormatProperties` to map
      `VkFormatFeatureFlags` to internal flags
      - `kTexturable_Flag` means can be sampled with linear filtering
      - `kRenderable_Flag` means renderable with blending and implies
        `kTexturable_Flag`
      - `kBlitSrc_Flag` and `kBlitDst_Flag` mean blit src/dst
    - each `FormatInfo` also has an array of `ColorTypeInfo`
      - the mapping is one-to-many
      - `fColorType` is the `GrColorType`
      - `fTransferColorType` is the color type of the cpu memory
        - `onTransferPixelsTo` calls `copyBufferToImage` and expects the
          buffer to have the xfer color type
        - `onTransferPixelsFrom` and `onReadPixels` call `copyImageToBuffer`
          and expect the buffer to have the xfer color type
      - `fFlags` describes the color type's intensions
        - `kUploadData_Flag` means it can be uploaded into
        - `kRenderable_Flag` means it can be renderered into
        - `kWrappedOnly_Flag`
      - `fReadSwizzle` is the glsl swizzle applied to the sampled value
      - `fWriteSwizzle` is the glsl swizzle applied to the output value
  - `GrCaps::getDefaultBackendFormat` picks the vk format for a color type
    - `onGetDefaultBackendFormat` uses `fColorTypeToFormatTable` to make color
      type to vk format
    - `isFormatTexturable` checks against `kTexturable_Flag`
    - `onAreColorTypeAndFormatCompatible` checks the color type and the format
      are compat
    - `supportedWritePixelsColorType`
    - `isFormatAsColorTypeRenderable` checks `kRenderable_Flag` of both the
      format and the color type
