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
