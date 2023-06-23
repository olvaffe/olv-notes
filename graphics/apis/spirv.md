SPIR-V
======

## Tools

- spec
  - <https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html>
- SPIRV-Headers
  - <https://github.com/KhronosGroup/SPIRV-Headers>
  - `spirv.h` defines enums for opcodes, scopes, semantics, etc.
- SPIRV-Tools
  - <https://github.com/KhronosGroup/SPIRV-Tools>
  - depends on SPIRV-Headers
  - provides tools (assembler, disassembler, optimizer, linker, etc.) and
    `libSPIRV-Tools.a` for working with SPIR-V
- glslang
  - <https://github.com/KhronosGroup/glslang>
  - depends on SPIRV-Tools
  - provides `glslangValidator` to convert GLSL/HLSL to SPIR-V
- SPIRV-Cross
  - <https://github.com/KhronosGroup/SPIRV-Cross>
  - provides `spirv-cross` to convert SPIR-V to GLSL/HLSL/MSL
- shaderc
  - <https://github.com/google/shaderc>
  - depends on glslang and SPIRV-Tools
  - provides tool (`glslc`) and library (`libshaderc`) to convert GLSL/HLSL
    to SPIR-V
- `glslangValidator`
  - `glslangValidator -V -o a.spirv a.frag`
  - `-x` to output something for inclusion by C source code
  - `-g` to include debug info
  - `-H` to print human-readable form of spirv

## Basics

- `cat a.frag`

    #version 460 core
    layout(location = 0) in mediump vec4 in_val;
    layout(location = 0) out vec4 out_val;
    void main() { out_val = in_val / 2.0; }
- `glslangValidator -V -H -o a.spirv a.frag`, or better yet, complie with `-g`
  and disassemble with `spirv-dis`

    ; SPIR-V
    ; Version: 1.0
    ; Generator: Khronos Glslang Reference Front End; 10
    ; Bound: 17
    ; Schema: 0
                   OpCapability Shader
              %2 = OpExtInstImport "GLSL.std.450"
                   OpMemoryModel Logical GLSL450
                   OpEntryPoint Fragment %main "main" %out_val %in_val
                   OpExecutionMode %main OriginUpperLeft
              %1 = OpString "a.frag"
                   OpSource GLSL 460 %1 "// OpModuleProcessed client vulkan100
    // OpModuleProcessed target-env vulkan1.0
    // OpModuleProcessed entry-point main
    #line 1
    #version 460 core
    
    layout(location = 0) in mediump vec4 in_val;
    layout(location = 0) out vec4 out_val;
    
    void main() {
        out_val = in_val / 2.0;
    }
    "
                   OpName %main "main"
                   OpName %out_val "out_val"
                   OpName %in_val "in_val"
                   OpDecorate %out_val Location 0
                   OpDecorate %in_val RelaxedPrecision
                   OpDecorate %in_val Location 0
                   OpDecorate %13 RelaxedPrecision
                   OpDecorate %15 RelaxedPrecision
                   OpDecorate %16 RelaxedPrecision
           %void = OpTypeVoid
              %4 = OpTypeFunction %void
          %float = OpTypeFloat 32
        %v4float = OpTypeVector %float 4
    %_ptr_Output_v4float = OpTypePointer Output %v4float
        %out_val = OpVariable %_ptr_Output_v4float Output
    %_ptr_Input_v4float = OpTypePointer Input %v4float
         %in_val = OpVariable %_ptr_Input_v4float Input
        %float_2 = OpConstant %float 2
           %main = OpFunction %void None %4
              %6 = OpLabel
                   OpLine %1 7 0
             %13 = OpLoad %v4float %in_val
             %15 = OpCompositeConstruct %v4float %float_2 %float_2 %float_2 %float_2
             %16 = OpFDiv %v4float %13 %15
                   OpStore %out_val %16
                   OpReturn
                   OpFunctionEnd
- physical layout of a spir-v module
  - header
    - word 0: magic number 0x07230203
    - word 1: version
    - word 2: generator magic number
    - word 3: bound, all ids are smaller than the bound
    - word 4: reserved
  - header is followed by instructions, where each instruction is
    - word 0: word count and opcode of the instruction
    - depending on opcode, word 0 can be followed by
      - optional instruction type id
      - optional instruction result id
    - word X..(count-1): optional operands
- logical layout of a spir-v module
  - instructions must be in the following order
  - all `OpCapability` instructions
    - declare what caps the module uses
  - all `OpExtension` instructions
    - declare what exts the module uses
    - strings are 0-terminated and padded to word boundaries
  - all `OpExtInstImport` instructions
  - single required `OpMemoryModel`
  - all entrypoint decls, using `OpEntryPoint`
  - all execution-mode decls, using `OpExecutionMode` or `OpExecutionModeId`
  - these debug instructions in order
    - `OpString`, `OpSourceExtension`, `OpSource`, and `OpSourceContinued`
    - `OpName` and `OpMemberName`
    - `OpModuleProcessed`
  - all annotation instructions
    - `Op*Decorate*`
  - all of these
    - type instructions `OpType*`
    - constant instructions `OpConstant` or `OpSpec`
    - variable instructions `OpVariable` with non-function storage class
  - all function decls (for functions defined in another module)
    - same as function defs below, except without blocks
  - all function defs
    - `OpFunction`
    - `OpFunctionParameter`s
    - blocks, where each block is
      - started with `OpLabel`
      - all `OpVariable` must have a storage class of `Function`
      - ended with a termination instruction
    - `OpFunctionEnd`

## UBOs

- definition
  - `OpTypeStruct` to define the struct type
  - `OpTypePointer` to define a pointer type to the struct, with Uniform
    storage class
  - `OpVariable` to define a variable of the pointer type, with Uniform
    storage class as well
  - one `OpTypePointer` for each of the struct members, with Uniform storage
    class again
- load
  - `OpAccessChain` on the variable to get a pointer to a struct member
  - `OpLoad` to load

## Rounding

- Conversion Instructions
  - `OpConvertFToU` converts floating point to unsigned int numerically with
    RTZ
  - `OpConvertFToS` converts floating point to signed int numerically with RTZ
  - `OpConvertSToF` converts signed int to floating point numerically
  - `OpConvertUToF` converts unsigned int to floating point numerically
  - `OpUConvert` converts unsigned width with truncation or zero-extension
  - `OpSConvert` converts signed width with truncation or signed-extension
  - `OpFConvert` converts floating-point width numerically
  - more
- `FPRoundingMode` is very limited
  - The FPRoundingMode decoration must be applied only to a width-only
    conversion instruction whose only uses are Object operands of OpStore
    instructions storing through a pointer to a 16-bit floating-point object
    in the StorageBuffer, PhysicalStorageBuffer, Uniform, or Output Storage
    Classes.
  - basically only on `OpFConvert` to halfs what are used in `OpStore`
- `RoundingModeRTE` and `RoundingModeRTZ`
  - new capabilities and execution modes introduced in 1.4 or
    `SPV_KHR_float_controls`
  - apply to `OpEntryPoint`
  - they specify the default rounding mode for an entrypoint
  - they are ignored when an instruction has an implied rounding mode (e.g.,
    `OpConvertFToU`) or is decorated with `FPRoundingMode` (e.g.,
    `OpFConvert`)
