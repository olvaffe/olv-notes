SPIR-V
======

## Links

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

## Example

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

## Chapter 1. Introduction

- 1.1. Goals
- 1.2. Execution Environment and Client API
- 1.3. About This Document
- 1.4. Extendability
- 1.5. Debuggability
- 1.6. Design Principles
- 1.7. Static Single Assignment (SSA)
- 1.8. Built-In Variables
- 1.9. Specialization
- 1.10. Example

## Chapter 2. Specification

- 2.1. Language Capabilities
- 2.2. Terms
  - control flow
    - Block: a contiguous sequence of instructions that starts with
      `OpLabel` and ends with a block termination instruction such as
      `OpBranch` or `OpReturn`
    - CFG: a graph whose nodes are blocks and edges are branches
  - structured control flow
    - a header block that contains `OpLoopMerge` followed by `OpBranch*`
    - a back-edge block that points back to the header block
    - a continue target which "continue" jumps to
    - a merge block where the control flow converges
    - simply put,
      - the control flow enters from the header block
      - it branches to the back-edge block
      - it either branches back to the header block, to the continue target,
        or to the merge block
    - a single-block loop is a loop construct where the header block, the
      continue target, and the back-edge block are the same block
- 2.3. Physical Layout of a SPIR-V Module and Instruction
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
- 2.4. Logical Layout of a Module
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
- 2.5. Instructions
- 2.6. Entry Point and Execution Model
- 2.7. Execution Modes
- 2.8. Types and Variables
- 2.9. Function Calling
- 2.10. Extended Instruction Sets
- 2.11. Structured Control Flow
- 2.12. Specialization
- 2.13. Linkage
- 2.14. Relaxed Precision
- 2.15. Debug Information
- 2.16. Validation Rules
- 2.17. Universal Limits
- 2.18. Memory Model
- 2.19. Derivatives
- 2.20. Code Motion
- 2.21. Deprecation
- 2.22. Unified Specification
- 2.23. Uniformity

## Chapter 3. Binary Form

- 3.1. Magic Number
- 3.2. Source Language
- 3.3. Execution Model
- 3.4. Addressing Model
- 3.5. Memory Model
- 3.6. Execution Mode
- 3.7. Storage Class
  - used by
    - `OpTypePointer`
    - `OpVariable`
    - more
  - `UniformConstant`: GL uniform or CL constant memory
  - `Input`: Input from pipeline
  - `Uniform`: GL UBO
  - `Output`: Output to pipeline
  - `Workgroup`: GL shared qualifier or CL local memory
  - `CrossWorkgroup`: CL global memory
  - `Private`: Regular global memory
  - `Function`: Regular function memory.
  - `Generic`:
  - `PushConstant`: VK push-constant memory
  - `AtomicCounter`: Atomic counter-specific memory
  - `Image`: For holding image memory
  - `StorageBuffer` GL SSBO
- 3.8. Dim
- 3.9. Sampler Addressing Mode
- 3.10. Sampler Filter Mode
- 3.11. Image Format
- 3.12. Image Channel Order
- 3.13. Image Channel Data Type
- 3.14. Image Operands
- 3.15. FP Fast Math Mode
- 3.16. FP Rounding Mode
- 3.17. Linkage Type
- 3.18. Access Qualifier
- 3.19. Function Parameter Attribute
- 3.20. Decoration
- 3.21. BuiltIn
- 3.22. Selection Control
- 3.23. Loop Control
- 3.24. Function Control
- 3.25. Memory Semantics <id>
- 3.26. Memory Operands
- 3.27. Scope <id>
- 3.28. Group Operation
- 3.29. Kernel Enqueue Flags
- 3.30. Kernel Profiling Info
- 3.31. Capability
- 3.32. Ray Flags
- 3.33. Ray Query Intersection
- 3.34. Ray Query Committed Type
- 3.35. Ray Query Candidate Type
- 3.36. Fragment Shading Rate
- 3.37. FP Denorm Mode
- 3.38. FP Operation Mode
- 3.39. Quantization Mode
- 3.40. Overflow Mode
- 3.41. Packed Vector Format
- 3.42. Cooperative Matrix Operands
- 3.43. Cooperative Matrix Layout
- 3.44. Cooperative Matrix Use
- 3.45. Initialization Mode Qualifier
- 3.46. Host Access Qualifier
- 3.47. Load Cache Control
- 3.48. Store Cache Control
- 3.49. Instructions

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
