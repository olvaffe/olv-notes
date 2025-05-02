RDNA3 Instruction Set Architecture
==================================

## Chapter 1. Introduction

- 1.1. Terminology
  - a workgroup is a collection of waves
    - a wave is a collection of 32 or 64 work items
    - a work item is aka thread, lane, etc.
  - a shader engine is a collection of shader arrays
    - a shader array is a collection of WGPs (compute unit pairs)
    - a shader array is aka processor array
- 1.2. Hardware Overview
  - block diagram
    - memory controller
    - command processors
    - ultra-threaded dispatch processor
    - processor array consisting of WGPs, where each WGP consists of
      - CU pair
      - LDS (Local Data Share)
    - caches
      - L2 RW
      - L1 RO
      - Constant
      - Instruction
      - GDS (Global Data Share)
  - data paths
    - memory controller has direct access to
      - host cpu mmio
      - system memory
      - device memory
    - command processor has direct access to
      - L2 RW Cache
      - Global Data Share
    - processor array has direct access to
      - Global Data Share
      - L1 RO Cache
      - Instruction Cache
      - Constant Cache
  - data sharing between work items
    - each WGP (CU pair) has 128KB LDS shared by work items
    - each SA (shader array) has 4KB GDS shared by WGPs

## Chapter 2. Shader Concepts

- conceptually, the shader program (aka kernel) executes independently on each
  work item
  - SMEM instructions transfer data between memory and SGPRs
  - VMEM instructions transfer data between memory and VGPRs
  - SALU instructions operate on up to two operands, from SGPRs or literals
  - VALU instructions operate on up to three operands, from VGPRs, SGPRs, or
    literals
- 2.1. Wave32 and Wave64
  - a wave has 32 or 64 work items
  - how Wave64 works is to issue each VMEM and VALU instruction twice, for the
    lower half and the higher half of work items respectively
- 2.2. Shader Types
- 2.3. Work-groups
- 2.4. Shader Padding Requirement

## Chapter 3. Wave State

- 3.1. State Overview
- 3.2. Control State: PC and EXEC
- 3.3. Storage State: SGPR, VGPR, LDS
- 3.4. Wave State Registers
- 3.5. Initial Wave State

## Chapter 4. Shader Instruction Set

- 4.1. Common Instruction Fields

## Chapter 5. Program Flow Control

- 5.1. Program Control
- 5.2. Instruction Clauses
- 5.3. Send Message Types
- 5.4. Branching
- 5.5. Work-groups and Barriers
- 5.6. Data Dependency Resolution
- 5.7. ALU Instruction Software Scheduling

## Chapter 6. Scalar ALU Operations

- 6.1. SALU Instruction Formats
- 6.2. Scalar ALU Operands
- 6.3. Scalar Condition Code (SCC)
- 6.4. Integer Arithmetic Instructions
- 6.5. Conditional Move Instructions
- 6.6. Comparison Instructions
- 6.7. Bit-Wise Instructions
- 6.8. Access Instructions
- 6.9. Memory Aperture Query

## Chapter 7. Vector ALU Operations

- 7.1. Microcode Encodings
- 7.2. Operands
- 7.3. Instructions
- 7.4. 16-bit Math and VGPRs
- 7.5. Packed Math
- 7.6. Dual Issue VALU
- 7.7. Data Parallel Processing (DPP)
- 7.8. VGPR Indexing
- 7.9. Wave Matrix Multiply Accumulate (WMMA)

## Chapter 8. Scalar Memory Operations

- 8.1. Microcode Encoding
- 8.2. Dependency Checking
- 8.3. Scalar Memory Clauses and Groups
- 8.4. Alignment and Bounds Checking

## Chapter 9. Vector Memory Buffer Instructions

- 9.1. Buffer Instructions
- 9.2. VGPR Usage
- 9.3. Buffer Data
- 9.4. Buffer Addressing
- 9.5. Alignment
- 9.6. Buffer Resource

## Chapter 10. Vector Memory Image Instructions

- 10.1. Image Instructions
- 10.2. Image Opcodes with No Sampler
- 10.3. Image Opcodes with a Sampler
- 10.4. VGPR Usage
- 10.5. Image Resource
- 10.6. Image Sampler
- 10.7. Data Formats
- 10.8. Vector Memory Instruction Data Dependencies
- 10.9. Ray Tracing
- 10.10. Partially Resident Textures

## Chapter 11. Global, Scratch and Flat Address Space

- 11.1. Instructions
- 11.2. Addressing
- 11.3. Memory Error Checking
- 11.4. Data

## Chapter 12. Data Share Operations

- 12.1. Overview
- 12.2. Pixel Parameter Interpolation
- 12.3. VALU Parameter Interpolation
- 12.4. LDS Direct Load
- 12.5. Data Share Indexed and Atomic Access
- 12.6. Global Data Share
- 12.7. Alignment and Errors

## Chapter 13. Float Memory Atomics

- 13.1. Rounding
- 13.2. Denormals
- 13.3. NaN Handling
- 13.4. Global Wave Sync & Atomic Ordered Count

## Chapter 14. Export: Position, Color/MRT

- 14.1. Pixel Shader Exports
- 14.2. Primitive Shader Exports (From GS shader stage)
- 14.3. Dependency Checking

## Chapter 15. Microcode Formats

- 15.1. Scalar ALU and Control Formats
- 15.2. Scalar Memory Format
- 15.3. Vector ALU Formats
- 15.4. Vector Parameter Interpolation Format
- 15.5. Parameter and Direct Load from LDS
- 15.6. LDS and GDS Format
- 15.7. Vector Memory Buffer Formats
- 15.8. Vector Memory Image Format
- 15.9. Flat Formats
- 15.10. Export Format

## Chapter 16. Instructions

- 16.1. SOP2 Instructions
- 16.2. SOPK Instructions
- 16.3. SOP1 Instructions
- 16.4. SOPC Instructions
- 16.5. SOPP Instructions
- 16.6. SMEM Instructions
  - `S_LOAD_B32` to `S_LOAD_B512` load 32 to 512 bits from the scalar data
    cache to sgprs
  - `S_BUFFER_LOAD_B32` to `S_BUFFER_LOAD_B512` load 32 to 512 bits from the
    scalar data cache to sgprs, using a buffer resource descriptor
- 16.7. VOP2 Instructions
- 16.8. VOP1 Instructions
- 16.9. VOPC Instructions
- 16.10. VOP3P Instructions
- 16.11. VOPD Instructions
- 16.12. VOP3 & VOP3SD Instructions
- 16.13. VINTERP Instructions
- 16.14. Parameter and Direct Load from LDS Instructions
- 16.15. LDS & GDS Instructions
- 16.16. MUBUF Instructions
  - `BUFFER_LOAD_FORMAT_X` to `BUFFER_LOAD_FORMAT_XYZW` load 1 to 4 components
    to vgprs, with format encoded in the buffer resource descriptor
  - `BUFFER_LOAD_U8` to `BUFFER_LOAD_B128` load 8 to 128 bits to vgprs
    - when less than 32 bits, the values are sign or zero extended
- 16.17. MTBUF Instructions
  - `TBUFFER_LOAD_FORMAT_X` to `TBUFFER_LOAD_FORMAT_XYZW` load 1 to 4
    components to vgprs, with format encoded in the instructions
- 16.18. MIMG Instructions
- 16.19. EXPORT Instructions
- 16.20. FLAT, Scratch and Global Instructions

## RDNA3 Instruction Set

- <https://gpuopen.com/amd-isa-documentation/>
- <https://www.amd.com/system/files/TechDocs/rdna3-shader-instruction-set-architecture-feb-2023_0.pdf>
- 3.3. Storage State: SGPR, VGPR, LDS
  - each wave is allocated 106 normal SGPRs
  - VGPRs are allocated in blocks of 16 for wave32 or 8 for wave64, and a
    shader may have up to 256 VGPRs.
- 16.1. SOP2 Instructions
  - `S_ADD_U32`
- 16.2. SOPK Instructions
  - `S_MOVK_I32`
- 16.3. SOP1 Instructions
  - `S_MOV_B32`
- 16.4. SOPC Instructions
  - `S_CMP_EQ_I32`
- 16.5. SOPP Instructions
  - `S_NOP`
- 16.6. SMEM Instructions
  - Scalar Memory Loads (SMEM) instructions allow a shader program to load
    data from memory into SGPRs through the Constant Cache ("Kcache").
    Instructions can load from 1 to 16 DWORDs. Data is loaded directly into
    SGPRs without any format conversion.
  - `S_LOAD_B32` reads 1 dword
    - addr is in base+offset
- 16.7. VOP2 Instructions
  - `V_CNDMASK_B32`
  - `V_ADD_NC_U32` Add two unsigned integers. No carry-in or carry-out.
- 16.8. VOP1 Instructions
  - `V_NOP`
- 16.9. VOPC Instructions
  - `V_CMP_F_F16`
- 16.10. VOP3P Instructions
  - `V_PK_MAD_I16`
- 16.11. VOPD Instructions
  - The VOPD encoded describes two VALU opcodes that are executed in
    parallel.
  - `V_DUAL_FMAC_F32`
- 16.12. VOP3 & VOP3SD Instructions
  - `V_NOP`
- 16.13. VINTERP Instructions
  - Parameter interpolation VALU instructions.
  - `V_INTERP_P10_F32`
- 16.14. Parameter and Direct Load from LDS Instructions
  - These instructions load data from LDS into a VGPR where the LDS address is
    derived from wave state and the M0 register.
  - `LDS_PARAM_LOAD`
- 16.15. LDS & GDS Instructions
  - This suite of instructions operates on data stored within the data share
    memory. The instructions transfer data between VGPRs and data share
    memory.
  - `DS_ADD_U32`
- 16.16. MUBUF Instructions
  - `BUFFER_LOAD_FORMAT_X`
- 16.17. MTBUF Instructions
  - `TBUFFER_LOAD_FORMAT_X`
- 16.18. MIMG Instructions
  - `IMAGE_LOAD`
- 16.19. EXPORT Instructions
  - Transfer vertex position, vertex parameter, pixel color, or pixel depth
    information to the output buffer
  - `EXP`
- 16.20.1. Flat Instructions
  - Flat instructions look at the per work-item address and determine for each
    work-item if the target memory address is in global, private or scratch
    memory.
  - `FLAT_LOAD_U8`
- 16.20.2. Scratch Instructions
  - Scratch instructions are like Flat, but assume all work-item addresses
    fall in scratch (private) space.
  - `SCRATCH_LOAD_U8`
- 16.20.3. Global Instructions
  - Global instructions are like Flat, but assume all work-item addresses fall
    in global memory space.
  - `GLOBAL_LOAD_U8`

## Vega Instruction Set

- <https://www.amd.com/system/files/TechDocs/vega-7nm-shader-instruction-set-architecture.pdf>
- 8.1.8. Buffer Resource
  - The buffer resource describes the location of a buffer in memory and the
    format of the data in the buffer. It is specified in four consecutive
    SGPRs (four aligned SGPRs) and sent to the texture cache with each buffer
    instruction.
- 12.1. SOP2 Instructions
- 12.2. SOPK Instructions
  - `S_MOVK_I32`
- 12.3. SOP1 Instructions
  - `S_MOV_B32`
- 12.4. SOPC Instructions
- 12.5. SOPP Instructions
  - `S_ENDPGM` End of program; terminate wavefront.
  - `S_WAITCNT` Wait for the counts of outstanding lds, vector-memory and
    export/vmem-write-data to be at or below the specified levels.
  - `S_SETPRIO`  User settable wave priority
- 12.6. SMEM Instructions
  - Scalar Memory Read (SMEM) instructions allow a shader program to load data
    from memory into SGPRs through the Scalar Data Cache, or write data from
    SGPRs to memory through the Scalar Data Cache.
    - `SBASE` SGPR-pair which provides a base address
    - `OFFSET` offset
  - `S_LOAD_DWORD`  Read 1 dword from scalar data cache
  - `S_LOAD_DWORDX4` 
  - `S_BUFFER_LOAD_B32` Read 1 dword from scalar data cache.
- 12.7. VOP2 Instructions
  - `V_ADD_U32`
  - `V_AND_B32`
  - `V_MUL_F32` Multiply two single-precision values.
- 12.7.1. VOP2 using VOP3 encoding
  - `V_ADD_U32` with alternative encoding for extra controls
- 12.8. VOP1 Instructions
  - `V_CVT_F32_I32` Convert from a signed integer to a single-precision float.
- 12.9. VOPC Instructions
- 12.10. VOP3P Instructions
- 12.11. VINTERP Instructions
- 12.12. VOP3A & VOP3B Instructions
  - `V_MED3_F32` Return median single precision value of three inputs.
  - `V_LSHL_ADD_U32`: `D.u = (S0.u << S1.u[4:0]) + S2.u.`
- 12.13. LDS & GDS Instructions
- 12.14. MUBUF Instructions
  - Buffer instructions (MTBUF and MUBUF) allow the shader program to read
    from, and write to, linear buffers in memory.
  - `BUFFER_LOAD_DWORD` Untyped buffer load dword
    - `IDXEN` Supply an index from VGPR
    - `OFFEN` Supply an offset from VGPR
- 12.15. MTBUF Instructions
- 12.16. MIMG Instructions
- 12.17. EXPORT Instructions
  - `EXP` Transfer vertex position, vertex parameter, pixel color, or pixel
    depth information to the output buffer. 
- 12.18.1. Flat Instructions
- 12.18.2. Scratch Instructions
- 12.18.3. Global Instructions
