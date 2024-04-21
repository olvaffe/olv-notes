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

## SPIRV-Reflect

- build
  - `cmake -S. -Bout -GNinj -DSPIRV_REFLECT_STATIC_LIB=ON`
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
