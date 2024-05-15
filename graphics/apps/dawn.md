Dawn
====

## WebGPU

- <https://github.com/gpuweb/gpuweb>
- specs
  - webgpu, <https://gpuweb.github.io/gpuweb/>
  - wgsl, <https://gpuweb.github.io/gpuweb/wgsl/>
- build
  - `git clone https://dawn.googlesource.com/dawn`
  - `python tools/fetch_dawn_dependencies.py --use-test-deps`
  - `cmake -S. -Bout -GNinja`

## Build System

- `tools/fetch_dawn_dependencies.py --use-test-deps` fetches
  - `third_party/abseil-cpp`
    - <https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/third_party/abseil-cpp/>
  - `third_party/glfw`
  - `third_party/jinja2`
    - <https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/third_party/jinja2>
  - `third_party/khronos/EGL-Registry`
  - `third_party/khronos/OpenGL-Registry`
  - `third_party/markupsafe`
    - <https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/third_party/markupsafe>
  - `third_party/vulkan-deps`
    - <https://chromium.googlesource.com/vulkan-deps/+/refs/heads/main/>
  - `third_party/googletest`
- cmake
  - `add_subdirectory(third_party)` builds third-party dependencies
    - `SPIRV-Headers`
    - `SPIRV-Tools`
    - `glslang`, because of `TINT_BUILD_CMD_TOOLS`/etc
    - `glfw`, because of `DAWN_USE_GLFW`
    - `libabsl`
    - `Vulkan-Headers`
  - `add_subdirectory(src/tint)` builds tint, WGSL transpiler
    - it supports spirv and wgsl as input langs
    - it supports spirv, wgsl, hlsl, glsl, and msl as output langs
    - `tint_info` prints shader info
      - it understands `.wgsl`, `.spv` (binary), and `.spvasm` (disassembly)
      - note that it only understands wgsl-subset of spirv
    - `tint` is offline transpiler
      - input lang is determined by the filename suffix
      - output lang is specified by `-f`
  - `add_subdirectory(generator)` defines the code generators
  - `add_subdirectory(src/dawn)` is dawn
    - `dawn_json_generator.py` generates various headers and `dawn_proc`
      library from `dawn.json` and `dawn_wire.json`
    - `add_subdirectory(partition_alloc)` provides `raw_ptr<T>`
    - `add_subdirectory(common)`
      - `dawn_version_generator.py` generates `Version_autogen.h`
        - it defines `dawn::kDawnVersion`
      - `dawn_gpu_info_generator.py` generates `GPUInfo_autogen.h` from
        `gpu_info.json`
        - it provides `IsAMD`, `IsIntel`, `GetVendorName`, etc.
      - `dawn_common` library
    - `add_subdirectory(platform)`
      - `dawn_platform` library
    - `add_subdirectory(native)`
      - `dawn_native` library
    - `add_subdirectory(wire)` is for remote webgpu
      - `dawn_json_generator.py` generates `WireCmd_autogen.h` from
        `dawn.json` and `dawn_wire.json`
      - `dawn_wire` library
    - `add_subdirectory(utils)`
      - `dawn_utils` library
    - `add_subdirectory(glfw)`
      - `dawn_glfw` library for glfw integration
    - `add_subdirectory(samples)`
      - `HelloTriangle`
      - `DawnInfo`

## `HelloTriangle`

- `InitSample`
  - `--backend=` specifies the backend
    - `vulkan` selects `wgpu::BackendType::Vulkan`
    - default to `wgpu::BackendType::Undefined`
  - `--enable-toggle=` enables toggles
- `init`
  - `CreateCppDawnDevice` creates a `wgpu::Device`
    - `dawnProcSetProcs` picks between `dawn::native::GetProcs()` or
      `dawn::wire::GetProcs()`
    - `wgpu::CreateInstance` creates a `wgpu::Instance`
    - `instance.RequestAdapter` creates a `wgpu::Adapter`
    - `adapter.RequestDevice` creates a `wgpu::Device`
    - `wgpu::glfw::CreateSurfaceForWindow` creates a `wgpu::Surface` from a
      `GLFWwindow`
    - `device.CreateSwapChain` creates a `wgpu::SwapChain`
  - `device.GetQueue` returns a `wgpu::Queue`
  - `dawn::utils::CreateBufferFromData` creates a `wgpu::Buffer`, and uploads
    data
  - `dawn::utils::CreateShaderModule` creates a `wgpu::ShaderModule` from wgsl
  - `device.CreateRenderPipeline` creates a `wgpu::RenderPipeline`
- `frame`
  - `swapchain.GetCurrentTextureView` returns a `wgpu::TextureView`
  - `device.CreateCommandEncoder` creates a `wgpu::CommandEncoder`
  - `encoder.BeginRenderPass` returns a `wgpu::RenderPassEncoder`
    - `pass.End` ends a render pass
  - `encoder.Finish` returns a `wgpu::CommandBuffer`
  - `queue.Submit` submits a `wgpu::CommandBuffer`
  - `swapchain.Present` presents
