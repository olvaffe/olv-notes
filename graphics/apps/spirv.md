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
- clspv
  - <https://github.com/google/clspv>
  - depends on clang, llvm, SPIRV-Headers, and SPIRV-Tools
  - provides tool (`clspv`) and library (`libclspv_core`) to convert opencl c
    to vk-ready spirv

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

## clspv

- build
  - `git clone https://github.com/google/clspv`
  - `python3 utils/fetch_sources.py`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DCLSPV_SHARED_LIB=ON`
    - `-DCMAKE_C_VISIBILITY_PRESET=hidden -DCMAKE_CXX_VISIBILITY_PRESET=hidden`
      - when `-DCLSPV_SHARED_LIB=ON`, it is desirable to hide llvm symbols
    - `-DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `ninja -C out clspv_core`
- the only entrypoint is `clspvCompileFromSourcesString`
  - compiler options
    - `-cl-std=CL3.0` selects CLC std
    - `-inline-entry-points` inlines all entrypoints
      - required for `__opencl_c_generic_address_space` feature
    - `-cl-single-precision-constant` treats double-precision float literals
      as single-precision
    - `-cl-kernel-arg-info` produces kernel argument info
    - `-rounding-mode-rte=16,32,64` sets `RoundingModeRTE`
    - `-rewrite-packed-structs` rewrites packed structs as i8 array
      - vk must support `storageBuffer8BitAccess`
    - `-std430-ubo-layout` uses more packed std430 layout
      - vk must support `uniformBufferStandardLayout`
    - `-decorate-nonuniform` decorates `NonUniform`
      - vk must support non-uniform descriptor indexing
    - `-hack-convert-to-float` adds a workaround for radv/aco
    - `-arch=spir` selects 32-bit pointers
    - `-spv-version=1.5` targets spirv 1.5
    - `-max-pushconstant-size=128` sets push contant size limit
      - it should match vk `maxPushConstantsSize`
    - `-max-ubo-size=16384`
      - it should match vk `maxUniformBufferRange`
    - `-global-offset` enables support for global offset (`get_global_offset`?)
    - `-long-vector` enables support for vectors of width 8 and 16
    - `-module-constants-in-storage-buffer` collects module-socped
      `__constants` into an ssbo
    - `-cl-arm-non-uniform-work-group-size` enables
      `cl_arm_non_uniform_work_group_size` extension

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
