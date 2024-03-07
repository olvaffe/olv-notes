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
- 16.17. MTBUF Instructions
- 16.18. MIMG Instructions
- 16.19. EXPORT Instructions
- 16.20. FLAT, Scratch and Global Instructions
