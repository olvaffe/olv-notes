SPIR-V Apps
===========

## Links

- <https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html>
  - spec
- <https://github.com/KhronosGroup/SPIRV-Registry>
  - extension registry
- <https://github.com/KhronosGroup/SPIRV-Guide>
  - tutorial
- <https://github.com/KhronosGroup/SPIRV-Headers>
  - header files
  - `spirv.h` defines enums for opcodes, scopes, semantics, etc.
- <https://github.com/KhronosGroup/SPIRV-Tools>
  - spirv assembler, disassembler, validator, and optimizer
  - depends on SPIRV-Headers
- <https://github.com/KhronosGroup/glslang>
  - glsl/hsl to spirv and glsl reflection
  - depends on SPIRV-Tools
  - provides `glslangValidator` to convert GLSL/HLSL to SPIR-V
- <https://github.com/KhronosGroup/SPIRV-LLVM-Translator>
  - bidirectional llvm ir and spirv translation
- <https://github.com/KhronosGroup/SPIRV-Cross>
  - spriv to glsl/hsl/msl and spirv reflection
  - provides `spirv-cross` to convert SPIR-V to GLSL/HLSL/MSL
- <https://github.com/KhronosGroup/SPIRV-Reflect>
  - spirv reflection
- <https://github.com/KhronosGroup/SPIRV-Visualizer>
  - live at <https://www.khronos.org/spir/visualizer/>
- shaderc
  - <https://github.com/google/shaderc>
  - depends on glslang and SPIRV-Tools
  - provides tool (`glslc`) and library (`libshaderc`) to convert GLSL/HLSL
    to SPIR-V

## glslang

- build
  - `git clone https://github.com/KhronosGroup/glslang`
  - `./update_glslang_sources.py`
  - `cmake -S . -B out -G Ninja -DCMAKE_BUILD_TYPE=Debug`
- `glslangValidator`
  - `glslangValidator -V -o a.spirv a.frag`
  - `-x` to output something for inclusion by C source code
  - `-g` to include debug info
  - `-H` to print human-readable form of spirv
- `glslang_input_t`
  - `glslang_source_t language` is the language
    - `GLSLANG_SOURCE_GLSL` or `GLSLANG_SOURCE_HLSL`
  - `glslang_stage_t stage` is the stage
    - one of `GLSLANG_STAGE_*`
  - `glslang_client_t client` is the glsl dialect
    - `GLSLANG_CLIENT_VULKAN` or `GLSLANG_CLIENT_OPENGL`
  - `glslang_target_client_version_t client_version` is the glsl dialect
    version
    - `GLSLANG_TARGET_VULKAN_*` or `GLSLANG_TARGET_OPENGL_*`
  - `glslang_target_language_t target_language` is the codegen target
    - only `GLSLANG_TARGET_SPV`
  - `glslang_target_language_version_t target_language_version` is the codegen
    target version
    - one of `GLSLANG_TARGET_SPV_1_*`
  - `int default_version` and `glslang_profile_t default_profile` are the
    default version line when none is specified
    - e.g., when `#version 460 core` is missing
    - when the client is vulkan, use `100` and `GLSLANG_NO_PROFILE`

## OpenCL C

- <https://github.com/KhronosGroup/OpenCL-Guide/blob/main/chapters/os_tooling.md>
- OpenCL C to LLVM IR
  - `clang -c -target spir -O0 -emit-llvm -o test.bc test.cl`
  - `-x` specifies the language explicitly
    - `c` for c
    - `c++` for c++
    - `cl` for opencl
    - `cuda` for cuda
    - otherwise, it is based on the filename suffix
  - `-std` specifies the language standard
    - `c17` for c17
    - `c++17` for c++17
    - `cl3.0` for opencl c 3.0
    - `cuda` for cuda
  - `-target` specifies the target arch
    - `x86_64-linux-gnu` for x86-64
    - `aarch64-linux-gnu` for aarch64
    - `spirv64-unknown-unknown` for spirv 64
      - it was called `spir64-unknown-unknown`
      - it invokes `llvm-spirv` to convert from llvm ir to spirv
  - `-emit-llvm` generates llvm ir instead
    - it cannot be used with linking so `-c` must also be specified
  - `-g` generates debug info
- LLVM IR to SPIR-V
  - `llvm-spirv test.bc -o test.spv`
- SPIR-V
  - `clCreateProgramWithIL` can load `test.spv`
  - the spirv generated this way is not compatible with vulkan
    - it uses capabilities that vulkan drivers do not support, such as
      - `OpCapability Addresses`
      - `OpCapability Linkage`
      - `OpCapability Kernel`
    - use `clspv` for that purpose instead
  - `spirv-dis test.spv` to disassemble
- SPIR-V to NIR
  - `spirv2nir test.spv`
    - must not include debug info
      - `Unsupported extension: OpenCL.DebugInfo.100`
    - must target `spirv32` rather than `spirv64`
  - `--optimize` to run basic optimizations

## SPIRV-Reflect

- build
  - `cmake -S. -Bout -GNinja -DSPIRV_REFLECT_STATIC_LIB=ON`
  - `ninja -C out spirv-reflect-static`
- meson

    cpp = meson.get_compiler('cpp')
    dep_spirv_reflect = cpp.find_library(
      'spirv-reflect-static',
      dirs: [spirv_reflect_path / 'out'],
      has_headers: ['spirv_reflect.h'],
      header_include_directories: include_directories(spirv_reflect_path),
    )

## SPIRV-Cross

- <https://github.com/KhronosGroup/SPIRV-Cross/wiki/Reflection-API-user-guide>
- table
  | GLSL declaration                | HLSL declaration                      | SPIRV-Cross             | Vulkan concept                      |
  | ------------------------------- | ------------------------------------- | ----------------------- | ----------------------------------- |
  | `sampler2D`                     | N/A                                   | `sampled_images`        | `COMBINED_IMAGE_SAMPLER`            |
  | `texture2D`                     | `Texture2D`                           | `separate_images`       | `SAMPLED_IMAGE`                     |
  | `image2D`                       | `RWTexture2D`                         | `storage_images`        | `STORAGE_IMAGE`                     |
  | `samplerBuffer`                 | `Buffer`                              | `separate_images`       | `UNIFORM_TEXEL_BUFFER`              |
  | `imageBuffer`                   | `RWBuffer`                            | `storage_images`        | `STORAGE_TEXEL_BUFFER`              |
  | `sampler`                       | `SamplerState`                        | `separate_samplers`     | `SAMPLER`                           |
  | `uniform UBO {}`                | `cbuffer buf {}`                      | `uniform_buffers`       | `UNIFORM_BUFFER(_DYNAMIC)`          |
  | `layout(push_constant) uniform` | N/A (root constants?)                 | `push_constant_buffers` | `VkPushConstantRange`               |
  | `subpassInput`                  | N/A                                   | `subpass_inputs`        | `INPUT_ATTACHMENT`                  |
  | `in vec2 uv (vs)`               | `void VertexMain (in uv : TEXCOORD0)` | `stage_inputs`          | `VkVertexInputAttributeDescription` |
  | `out vec4 FragColor (fs)`       | `float4 FragmentMain() : SV_Target0`  | `stage_outputs`         |                                     |
  | `buffer SSBO {}`                | `(RW)StructuredBuffer` / etc          | `storage_buffer`        | `STORAGE_BUFFER(_DYNAMIC)`          |
