SPIR-V
======

## Links

- spec
  - <https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html>
  - <https://registry.khronos.org/SPIR-V/specs/unified1/GLSL.std.450.html>
  - <https://registry.khronos.org/SPIR-V/specs/unified1/OpenCL.ExtendedInstructionSet.100.html>
- client apis
  - <https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_Env.html>
  - <https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap52.html>
- SPIRV-Headers
  - <https://github.com/KhronosGroup/SPIRV-Headers>
  - `spirv.h` defines enums for opcodes, scopes, semantics, etc.
- SPIRV-Tools
  - <https://github.com/KhronosGroup/SPIRV-Tools>
  - depends on SPIRV-Headers
  - provides tools (assembler, disassembler, optimizer, linker, etc.) and
    `libSPIRV-Tools.a` for working with SPIR-V

## Chapter 1. Introduction

- 1.1. Goals
- 1.2. Execution Environment and Client API
  - a spirv module is consumed by an execution environment, as specified by a
    client api
  - e.g., vulkan and opencl both specify their execution environments
- 1.3. About This Document
- 1.4. Extendability
  - `OpExtension` to enable extensions
- 1.5. Debuggability
- 1.6. Design Principles
  - regularity: instructions have the same binary format
  - non combinatorial: types are parameterized; operations are orthogonal to
    scalar/vector size (but distinguishes integers and floats)
  - modeless: there is no mode bits that affect semantics of operations
  - declarative: externally-visible side effects are declared, without
    requiring reflection
  - ssa: results of intermediate operations are strictly ssa; there are also
    load/store for variables
  - io: io is done through variable load/store
- 1.7. Static Single Assignment (SSA)
  - spirv supports phis and loads/stores for split control flow
- 1.8. Built-In Variables
  - spirv supports, for example, glsl built-in variables through variable
    decorations
- 1.9. Specialization
  - spirv supports specialization
- 1.10. Example
  - glsl fs

    #version 450

    in vec4 color1;
    in vec4 multiplier;
    noperspective in vec4 color2;
    out vec4 color;

    struct S {
        bool b;
        vec4 v[5];
        int i;
    };

    uniform blockName {
        S s;
        bool cond;
    };

    void main()
    {
        vec4 scale = vec4(1.0, 1.0, 2.0, 1.0);

        if (cond)
            color = color1 + s.v[2];
        else
            color = sqrt(color2) * scale;

        for (int i = 0; i < 4; ++i)
            color *= multiplier;
    }
  - spirv

    ; Magic:     0x07230203 (SPIR-V)
    ; Version:   0x00010000 (Version: 1.0.0)
    ; Generator: 0x00080001 (Khronos Glslang Reference Front End; 1)
    ; Bound:     63
    ; Schema:    0

                   OpCapability Shader
              %1 = OpExtInstImport "GLSL.std.450"
                   OpMemoryModel Logical GLSL450
                   OpEntryPoint Fragment %4 "main" %31 %33 %42 %57
                   OpExecutionMode %4 OriginLowerLeft

    ; Debug information
                   OpSource GLSL 450
                   OpName %4 "main"
                   OpName %9 "scale"
                   OpName %17 "S"
                   OpMemberName %17 0 "b"
                   OpMemberName %17 1 "v"
                   OpMemberName %17 2 "i"
                   OpName %18 "blockName"
                   OpMemberName %18 0 "s"
                   OpMemberName %18 1 "cond"
                   OpName %20 ""
                   OpName %31 "color"
                   OpName %33 "color1"
                   OpName %42 "color2"
                   OpName %48 "i"
                   OpName %57 "multiplier"

    ; Annotations (non-debug)
                   OpDecorate %15 ArrayStride 16
                   OpMemberDecorate %17 0 Offset 0
                   OpMemberDecorate %17 1 Offset 16
                   OpMemberDecorate %17 2 Offset 96
                   OpMemberDecorate %18 0 Offset 0
                   OpMemberDecorate %18 1 Offset 112
                   OpDecorate %18 Block
                   OpDecorate %20 DescriptorSet 0
                   OpDecorate %42 NoPerspective

    ; All types, variables, and constants
              %2 = OpTypeVoid
              %3 = OpTypeFunction %2                      ; void ()
              %6 = OpTypeFloat 32                         ; 32-bit float
              %7 = OpTypeVector %6 4                      ; vec4
              %8 = OpTypePointer Function %7              ; function-local vec4*
             %10 = OpConstant %6 1
             %11 = OpConstant %6 2
             %12 = OpConstantComposite %7 %10 %10 %11 %10 ; vec4(1.0, 1.0, 2.0, 1.0)
             %13 = OpTypeInt 32 0                         ; 32-bit int, sign-less
             %14 = OpConstant %13 5
             %15 = OpTypeArray %7 %14
             %16 = OpTypeInt 32 1
             %17 = OpTypeStruct %13 %15 %16
             %18 = OpTypeStruct %17 %13
             %19 = OpTypePointer Uniform %18
             %20 = OpVariable %19 Uniform
             %21 = OpConstant %16 1
             %22 = OpTypePointer Uniform %13
             %25 = OpTypeBool
             %26 = OpConstant %13 0
             %30 = OpTypePointer Output %7
             %31 = OpVariable %30 Output
             %32 = OpTypePointer Input %7
             %33 = OpVariable %32 Input
             %35 = OpConstant %16 0
             %36 = OpConstant %16 2
             %37 = OpTypePointer Uniform %7
             %42 = OpVariable %32 Input
             %47 = OpTypePointer Function %16
             %55 = OpConstant %16 4
             %57 = OpVariable %32 Input

    ; All functions
              %4 = OpFunction %2 None %3                  ; main()
              %5 = OpLabel
              %9 = OpVariable %8 Function
             %48 = OpVariable %47 Function
                   OpStore %9 %12
             %23 = OpAccessChain %22 %20 %21              ; location of cond
             %24 = OpLoad %13 %23                         ; load 32-bit int from cond
             %27 = OpINotEqual %25 %24 %26                ; convert to bool
                   OpSelectionMerge %29 None              ; structured if
                   OpBranchConditional %27 %28 %41        ; if cond
             %28 = OpLabel                                ; then
             %34 = OpLoad %7 %33
             %38 = OpAccessChain %37 %20 %35 %21 %36      ; s.v[2]
             %39 = OpLoad %7 %38
             %40 = OpFAdd %7 %34 %39
                   OpStore %31 %40
                   OpBranch %29
             %41 = OpLabel                                ; else
             %43 = OpLoad %7 %42
             %44 = OpExtInst %7 %1 Sqrt %43               ; extended instruction sqrt
             %45 = OpLoad %7 %9
             %46 = OpFMul %7 %44 %45
                   OpStore %31 %46
                   OpBranch %29
             %29 = OpLabel                                ; endif
                   OpStore %48 %35
                   OpBranch %49
             %49 = OpLabel
                   OpLoopMerge %51 %52 None               ; structured loop
                   OpBranch %53
             %53 = OpLabel
             %54 = OpLoad %16 %48
             %56 = OpSLessThan %25 %54 %55                ; i < 4 ?
                   OpBranchConditional %56 %50 %51        ; body or break
             %50 = OpLabel                                ; body
             %58 = OpLoad %7 %57
             %59 = OpLoad %7 %31
             %60 = OpFMul %7 %59 %58
                   OpStore %31 %60
                   OpBranch %52
             %52 = OpLabel                                ; continue target
             %61 = OpLoad %16 %48
             %62 = OpIAdd %16 %61 %21                     ; ++i
                   OpStore %48 %62
                   OpBranch %49                           ; loop back
             %51 = OpLabel                                ; loop merge point
                   OpReturn
                   OpFunctionEnd

## Chapter 2. Specification

- 2.1. Language Capabilities
  - a spirv module declares used features through `OpCapability`
  - a client api must support the used features
- 2.2. Terms
  - instructions
    - `Instruction`
      - `WordCount`
      - op code
      - optional `Result <id>`
      - optional `<id>` for the instruction type
      - zero or more `Operand` that are `<id>`s or `literal`s
    - `Non-Semantic Instruction` are instructions that have no semantic impact
      and can be safely ignored/removed
  - objects
    - intermediate object/value/result: `Result <id>` of an operation
    - memory object: created through `OpVariable`
  - types
    - boolean
    - numeric
      - integer
      - floating-point
    - scalar
    - composite
      - vector, formed from scalars
      - matrix, formed from vectors
      - aggregate
        - array, whose elements are of the same type
        - struct, whose members are of various type
    - opaque
      - image, sampler, and sampled image
      - more
    - pointer
      - physical: `OpTypePointer`
      - logical
  - module
    - module: can have multiple entry points but only one set of capabilities
    - entry point: `OpEntryPoint`
      - execution model: a graphical stage or a cl kernel
      - execution mode: affect interaction with the execution environment
  - control flow
    - Block: a contiguous sequence of instructions that starts with
      `OpLabel` and ends with a block termination instruction such as
      `OpBranch` or `OpReturn`
    - CFG: a graph whose nodes are blocks and edges are branches
    - Invocation: a single execution of an entry point in a module
      - e.g., an invocation operates on a single work item in compute and on a
        single vertex in vs
    - Quad and Subgroup: a partition of invocations
      - e.g., invocations are partitioned into subgroups in compute and into
        quads (4 fragments) in fs
    - Invocation Group: all invocations of a particular operation
      - e.g., all invocations of a workgroup in compute, or of a primitive or
        draw in graphics stages
    - Dynamic Instance: within a single invocation, a single instruction can
      be executed multiple times, generating multiple dynamic instances of the
      instruction
      - e.g., the instruction is executed in a loop
      - two invocations execute the same dynamic instances as long as they
        follow the same control flow path
    - Dynamically Uniform: an `<id>` is dynamically uniform for a dynamic
      instance consuming it if its value is the same for all invocations (in
      the invocation group) that execute that dynamic instance
    - Uniform Control Flow: a uniform/converged control flow occurs at an
      instruction when all invocations (in the invocation group) execute the
      same dynamic instances of the instruction
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
  - most instructions create `Result <id>`
  - instructions refer to `Result <id>` created by other instructions via
    `<id>` operands
  - variable access
    - `OpVariable` to allocate an object in memory and create a `Result <id>`
      that is the name of a pointer to it
    - `OpAccessChain` to create a pointer to a subpart of a composite object
      in memory
    - `OpLoad` through a pointer, giving the loaded object a `Result <id>`
    - `OpStore` through a pointer, to write a value. There is no `Result <id>`
- 2.6. Entry Point and Execution Model
  - `OpEntryPoint` declares an entry point which consists of
    - Execution Model
    - `<id>` of `OpFunction`
    - name, for external reference
    - `<id>`s of global `OpVariable`
- 2.7. Execution Modes
  - `OpExecutionMode` declares execution modes for an entry point
- 2.8. Types and Variables
  - types are built bottom up, using `OpType*`
  - types (and others) can be decorated with `Op*Decorate*`
  - variables are allocated using `OpVariable`, which requires a pointer type
    and a storage class
- 2.9. Function Calling
  - `OpFunctionCall` calls a `OpFunction`
- 2.10. Extended Instruction Sets
  - `OpExtInstImport` imports an extended instruction set, such as
    `GLSL.std.450` for glsl built-in functions
  - `OpExtInst` calls an extended instruction
- 2.11. Structured Control Flow
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
- 2.12. Specialization
  - `OpSpecConstant*`
- 2.13. Linkage
  - functions an global variables of a module are private by default
  - they can be decorated with `LinkageAttributes`, to export or import
- 2.14. Relaxed Precision
  - `RelaxedPrecision` allows 32-bit integer/float operations to execute as
    16-bit integer/float operations
  - when applied to a variable, function parameter, or struct member, all
    loads/stores may be treated as though they were decorated with
    `RelaxedPrecision`
  - when applied to (the result id of) an instruction, the instruction is to
    operate at relaxed precision
- 2.15. Debug Information
  - `OpSource` for source language
  - `OpName` and `OpMemberName` for object names
  - `OpLine` and `OpNoLine` for line numbers
- 2.16. Validation Rules
- 2.17. Universal Limits
- 2.18. Memory Model
  - `OpMemoryModel` specifies the addressing model and the memory model
- 2.19. Derivatives
- 2.20. Code Motion
- 2.21. Deprecation
- 2.22. Unified Specification
- 2.23. Uniformity
  - a `Result <id>` decorated as `Uniform` or `UniformId` means all
    invocations in the specified scope compute the same value for any given
    dynamic instance of the instruction
    - e.g., impl can use the hint to store the value in sgpr rather than vgpr

## Chapter 3. Binary Form

- 3.1. Magic Number
  - `0x07230203`
- 3.2. Source Language
  - `GLSL`, `OpenCL_C`, `HLSL`, etc.
- 3.3. Execution Model
  - `Vertex`, `Fragment`, `GLCompute`, `Kernel`, etc.
- 3.4. Addressing Model
  - `Logical`, `Physical32`, `Physical64`, etc.
- 3.5. Memory Model
  - `GLSL450`, `OpenCL`, `Vulkan`
- 3.6. Execution Mode
  - `RoundingModeRTE`, `RoundingModeRTZ`
    - if missing, the default rounding mode is defined by client apis
  - `Invocations`, `InputPoints`, `InputLines`, etc.
    - gs-specific
  - `LocalSize`, etc.
    - compute-specific
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
  - `1D`, `2D`, `3D`, `Cube`, etc.
- 3.9. Sampler Addressing Mode
  - `None`, `ClampToEdge`, etc.
- 3.10. Sampler Filter Mode
  - `Nearest`, `Linear`
- 3.11. Image Format
  - `Unknown`, `Rgba32f`, etc.
- 3.12. Image Channel Order
  - `R`, `RG`, `RGB`, etc.
- 3.13. Image Channel Data Type
  - `SnormInt8`, `SnormInt16`, `UnormInt8`, etc.
- 3.14. Image Operands
  - `Bias`, `Lod`, `Grad`, etc.
- 3.15. FP Fast Math Mode
  - `NotNaN`, `NotInf`, etc.
- 3.16. FP Rounding Mode
  - `RTE`, `RTZ`, etc.
- 3.17. Linkage Type
  - `Export`, `Import`, etc.
- 3.18. Access Qualifier
  - `ReadOnly`, `WriteOnly`, etc.
- 3.19. Function Parameter Attribute
  - `Zext`, `Sext`, `ByVal`, etc.
- 3.20. Decoration
  - `RelaxedPrecision` allows reduced precision operations
  - `BuiltIn` indicates a built-in variable
- 3.21. BuiltIn
  - vs
    - `Position`, `PointSize`, etc.
  - compute
    - `WorkgroupSize`, `WorkgroupId`, etc.
- 3.22. Selection Control
  - `Flatten`, `DontFlatten`
- 3.23. Loop Control
  - `Unroll`, `DontUnroll`, etc.
- 3.24. Function Control
  - `Inline`, `DontInline`, etc.
- 3.25. `Memory Semantics <id>`
  - `Acquire`, `Release`, etc.
- 3.26. Memory Operands
  - `Volatile`, `Aligned`, etc.
- 3.27. `Scope <id>`
  - `CrossDevice`, `Device`, `Workgroup`, etc.
- 3.28. Group Operation
  - `Reduce`, `InclusiveScan`, etc.
- 3.29. Kernel Enqueue Flags
  - `NoWait`, `WaitKernel`, etc.
- 3.30. Kernel Profiling Info
  - `CmdExecTime`
- 3.31. Capability
  - `Matrix`, `Shader`, `Float16`, etc.
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
  - `PackedVectorFormat4x8Bit`
- 3.42. Cooperative Matrix Operands
- 3.43. Cooperative Matrix Layout
- 3.44. Cooperative Matrix Use
- 3.45. Initialization Mode Qualifier
- 3.46. Host Access Qualifier
- 3.47. Load Cache Control
- 3.48. Store Cache Control
- 3.49. Instructions
  - Miscellaneous
    - `OpNop`, `OpUndef`, `OpSizeOf`, etc.
  - Debug
    - `OpSource`, `OpName`, `OpLine`, etc.
  - Annotation
    - `OpDecorate`, `OpMemberDecorate`, etc.
  - Extension
    - `OpExtension`
    - `OpExtInstImport`, `OpExtInst`
  - Mode-Setting
    - `OpMemoryModel`
    - `OpEntryPoint`, `OpExecutionMode`
    - `OpCapability`
  - Type-Declaration
    - `OpTypeVoid`, `OpTypeBool`, `OpTypeInt`, `OpTypeFloat`
    - `OpTypeVector`, `OpTypeMatrix`
    - `OpTypeImage`, `OpTypeSampler`, `OpTypeSampledImage`
    - `OpTypeArray`, `OpTypeRuntimeArray`, `OpTypeStruct`, `OpTypeOpaque`
    - `OpTypePointer`, `OpTypeFunction`
  - Constant-Creation
    - `OpConstant`, `OpConstantComposite`, etc.
  - Memory
    - `OpVariable`, `OpLoad`, `OpStore`, `OpAccessChain`, etc.
  - Function
    - `OpFunction`, `OpFunctionParameter`, `OpFunctionEnd`
  - Image
    - `OpImageSample*`, `OpImageGather`
    - `OpImageFetch`
    - `OpImageRead`, `OpImageWrite`
    - `OpImageQuery*`
  - Conversion
    - `OpConvertFToU` with RTZ
    - `OpConvertFToS` with RTZ
    - `OpConvertSToF`
    - `OpConvertUToF`
    - `OpUConvert` with truncation or zero-extension
    - `OpSConvert` with truncation or sign-extension
    - `OpFConvert`
  - Composite
  - Arithmetic
    - `OpIAdd`, `OpFAdd`, etc.
  - Bit
    - `OpShiftRightLogical`, `OpShiftRightArithmetic`, `OpBitwiseOr`, etc.
  - Relational and Logical
    - `OpAny`, `OpAll`, `OpLogicalEqual`, `OpLogicalOr`
  - Derivative
    - `OpDPdx`, `OpDPdy`, etc.
  - Control-Flow
    - `OpPhi`, `OpLoopMerge`, `OpLabel`, `OpBranch`, etc.
  - Atomic
    - `OpAtomicLoad`, `OpAtomicStore`, etc.
  - Primitive
    - `OpEmitVertex`, `OpEndPrimitive`, etc.
  - Barrier
  - Group and Subgroup
    - `OpGroupAll`, `OpGroupAny`, `OpGroupBroadcast`, etc.
  - Device-Side Enqueue
  - Pipe
  - Non-Uniform

## UBOs/SSBOs

- definition
  - `OpTypeStruct` to define the struct type
  - `OpTypePointer` to define a pointer type to the struct
    - `Uniform` storage class for UBOs
    - `StorageBuffer` storage class for SSBOs
  - `OpVariable` to define a variable of the pointer type, with corresponding
    storage classes
  - one `OpTypePointer` for each of the struct members, with corresponding
    storage classes
- load/store
  - `OpAccessChain` on the variable to get a pointer to a struct member
    specified by the indices
  - `OpLoad` to load
  - `OpStore` to store for SSBOs

## TBOs/IBOs

- definition
  - `OpTypeImge` to define the image type
    - typed because TBOs are `gtextureBuffer` and IBOs are `gimageBuffer`
    - while spirv allows untyped, `VUID-StandaloneSpirv-OpTypeImage-04656` requires typed
      - `RelaxedPrecision` can be applied to a sampling instruction and to the
        variable holding the result of a sampling instruction
  - `OpTypePointer` to define a pointer type to the struct, with
    `UniformConstant` storage class
  - `OpVariable` to define a variable of the pointer type, with
    `UniformConstant` storage class
- load/store
  - `OpLoad` the variable to get the image handle
  - `OpImageFetch` the image handle to load for TBOs
  - `OpImageRead` the image handle to load for IBOs
  - `OpImageWrite` the image handle to store for IBOs

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
