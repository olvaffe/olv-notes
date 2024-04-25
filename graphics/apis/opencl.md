OpenCL
======

## History

- 1.0 was released in 2008
- 1.1 was released in 2010
- 1.2 was released in 2011
- 2.0 was released in 2013
  - SVM, shared virtual memory
- 2.1 was released in 2015
  - "OpenCL C++ Kernel Language" based on C++14
  - SPIR-V
- 2.2 was released in 2017
- 3.0 was released in 2020
  - only OpenCL 1.2 functionality is mandatory
  - all OpenCL 2.x and OpenCL 3.0 functionality is optional
  - replaced "OpenCL C++ Kernel Language" by "C++ for OpenCL", based on C++17

## References

- specifications
  - <https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_API.html>
    - this defines the core api
  - <https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_C.html>
    - this defines opencl c language
  - <https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_Ext.html>
    - this defines all `cl_khr_*` extensions
  - <https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_Env.html>
- http://www.nvidia.com/content/CUDA-ptx_isa_1.4.pdf (how NVIDIA does it in their tgsi equivalent),
- http://developer.amd.com/gpu_assets/Intermediate_Language_Specification--Stream_Processor.pdf (how AMD does it in their tgsi equivalent),
- http://llvm.org/devmtg/2009-10/OpenCLWithLLVM.pdf (generic presentation about how Apple did their OpenCL Clang integration)
- http://llvm.org/docs/CodeGenerator.html (how LLVM code generators work) 

## Chapter 1. Introduction

- 1.1. Normative References
- 1.2. Version Numbers
- 1.3. Unified Specification

## Chapter 2. Glossary

## Chapter 3. The OpenCL Architecture

- 3.1. Platform Model
- 3.2. Execution Model
  - ND-range
    - `work_dim` specifies the dimension
    - `global_work_offset` specifies the offset in each dimension
    - `global_work_size` specifies the size in each dimension
    - `local_work_size` specifies the size of a work-group in each dimension
    - e.g., let's say we have `float[64]` and we want to copy `float[16..47]`
      one by one.  The most straightforward way is to
      - set `work_dim` to 1, because the array is 1-dimensional
      - set `global_work_offset` to 16, because it starts at `float[16]`
      - set `global_work_size` to 32, because we want to copy 32 floats
      - leave `local_work_size` implementation-determined, because each copy
        is independent and we don't care about the size of a work-group
      - the kernel can just `dst[get_global_id()] = src[get_global_id()]`
- 3.3. Memory Model
  - Host Memory
    - system ram
  - Device Memory
    - `global`
      - device ram
    - `constant`
      - also device ram?
      - read-only
    - `local`
      - where is this?
    - `private`
      - register file
- 3.4. The OpenCL Framework

## Chapter 4. The OpenCL Platform Layer

- 4.1. Querying Platform Info
- 4.2. Querying Devices
  - `clGetDeviceInfo`
    - `CL_DEVICE_MAX_COMPUTE_UNITS`
    - `CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS`, at least 3 and normally 3
    - `CL_DEVICE_MAX_WORK_ITEM_SIZES`, max number of work-items in each
      dimension in a work-group
    - `CL_DEVICE_MAX_WORK_GROUP_SIZE`, max number of work-items in a work-group
    - `CL_DEVICE_MAX_PARAMETER_SIZE` at least 1024 bytes (full) or 256 bytes
      (embedded)
      - push constants?
    - `CL_DEVICE_GLOBAL_MEM_CACHELINE_SIZE`
    - `CL_DEVICE_GLOBAL_MEM_CACHE_SIZE`
    - `CL_DEVICE_GLOBAL_MEM_SIZE`
    - `CL_DEVICE_MAX_CONSTANT_BUFFER_SIZE` at least 64KB (full) or 1KB
      (embedded)
    - `CL_DEVICE_LOCAL_MEM_SIZE` at least 32KB (full) or 1KB (embedded)
    - `CL_DEVICE_MAX_NUM_SUB_GROUPS`
      - max queued waves
    - `CL_DEVICE_SUB_GROUP_INDEPENDENT_FORWARD_PROGRESS`
      - true
    - `CL_DEVICE_PREFERRED_WORK_GROUP_SIZE_MULTIPLE`
      - minimum wave size?
- 4.3. Partitioning a Device
- 4.4. Contexts

## Chapter 5. The OpenCL Runtime

- 5.1. Command-Queues
- 5.2. Buffer Objects
- 5.3. Image Objects
- 5.4. Pipes
- 5.5. Querying, Unmapping, Migrating, Retaining and Releasing Memory Objects
- 5.6. Shared Virtual Memory
- 5.7. Sampler Objects
- 5.8. Program Objects
  - math compiler options
    - `-cl-single-precision-constant` treats fp64 literals as fp32 ones; e.g.,
      `0.123` is treated as `0.123f`
    - `-cl-mad-enable` allows `a * b + c` to be replaced by `mad(a, b, c)`
    - `-cl-fast-relaxed-math` implies all unsafe math optimizations
- 5.9. Kernel Objects
  - `clGetKernelWorkGroupInfo`
    - `CL_KERNEL_WORK_GROUP_SIZE`
      - `CL_DEVICE_LOCAL_MEM_SIZE` divided by `CL_KERNEL_LOCAL_MEM_SIZE`?
    - `CL_KERNEL_LOCAL_MEM_SIZE`
      - where is local memory?
    - `CL_KERNEL_PREFERRED_WORK_GROUP_SIZE_MULTIPLE`
      - wave size?
    - `CL_KERNEL_PRIVATE_MEM_SIZE`
      - kernel register usage
  - `clGetKernelSubGroupInfo`
    - `CL_KERNEL_MAX_SUB_GROUP_SIZE_FOR_NDRANGE`
      - this is the wave size so probably 16, 32, 64, 128
    - `CL_KERNEL_SUB_GROUP_COUNT_FOR_NDRANGE`
      - work-group size divided by wave size?
    - `CL_KERNEL_LOCAL_SIZE_FOR_SUB_GROUP_COUNT`
      - ?
    - `CL_KERNEL_MAX_NUM_SUB_GROUPS`
      - register file size divided by kernel register usage?
- 5.10. Executing Kernels
  - `clEnqueueNDRangeKernel`
    - `work_dim`
      - number of dimensions
      - 1, 2, or 3
    - `global_work_size`
      - numbers of work-items in each dimension
      - total number must not exceed `sizeof(size_t)`
    - `local_work_size`
      - numbers of work-items that make up a work-group
      - `global_work_size[dim]` must be a multiple of `local_work_size[dim]`
      - total number must not exceed `CL_DEVICE_MAX_WORK_GROUP_SIZE`
      - each dimension must not exceed `CL_DEVICE_MAX_WORK_ITEM_SIZES[dim]`
- 5.11. Event Objects
- 5.12. Markers, Barriers and Waiting for Events
- 5.13. Out-of-order Execution of Kernels and Memory Object Commands
- 5.14. Profiling Operations on Memory Objects and Kernels
- 5.15. Flush and Finish

## Chapter 6. Associated OpenCL specification

- 6.1. SPIR-V Intermediate Language
- 6.2. Extensions to OpenCL
- 6.3. The OpenCL C Kernel Language

## Chapter 7. OpenCL Embedded Profile

## Appendix A: Host environment and thread safety

## Appendix B: Portability

## Appendix C: Application Data Types

## Appendix D: Checking for Memory Copy Overlap

## Appendix E: Changes to OpenCL

## Appendix F: Error Codes

## Appendix G: Other Miscellaneous Enums

## Appendix H: OpenCL 3.0 Backwards Compatibility

## The OpenCL C Specification

- Chapter 6. The OpenCL C Programming Language
  - 6.1. Unified Specification
  - 6.2. Optional functionality
  - 6.3. Supported Data Types
  - 6.4. Conversions and Type Casting
  - 6.5. Operators
  - 6.6. Vector Operations
  - 6.7. Address Space Qualifiers
  - 6.8. Access Qualifiers
  - 6.9. Function Qualifiers
  - 6.10. Storage-Class Specifiers
  - 6.11. Restrictions
  - 6.12. Preprocessor Directives and Macros
    - `#pragma OPENCL EXTENSION extensionname : behavior`
  - 6.13. Attribute Qualifiers
    - `__attribute__((opencl_unroll_hint(N)))`
      - there is also the non-standard `#pragma unroll N` and
        `#pragma nounroll`
  - 6.14. Blocks
  - 6.15. Built-in Functions
    - work-item functions
      - `get_work_dim` returns `work_dim`
      - `get_global_size` returns `global_work_size`
      - `get_global_id` returns an id between `[global_work_offset, global_work_offset + get_global_size - 1]`
      - `get_local_size` returns `local_work_size`
        - if not specified, this returns the implicit local work size picked by impl
      - `get_local_id` returns an id between `[0, local_work_size - 1]`
      - `get_num_groups` returns `global_work_size / global_local_size`
      - `get_group_id` returns an id between `[0, get_num_groups - 1]`
      - `get_global_offset` returns `global_work_offset`
    - math functions
      - `fma` returns `a * b + c` with one rounding
        - it is by definition more precise than `a * b + c` which rounds twice
        - with native hw support, it is one operation and can be twice as fast
          - for fp16 at least
          - for fp32, it can be extremely slow because it might need
            simulation
      - `mad` returns `a * b + c` with a twist
        - on full profile, it is the same as either `fma` or `a * b + c`
        - on embedded profile, it can be any value
        - with native hw support, it is one operation and can be twice as fast
      - `#pragma OPENCL FP_CONTRACT on-off-switch`
        - c11 states that, `#pragma STDC FP_CONTRACT ON` allows contracting of
          float-point expressions; that is, optimizations that omit rounding
          errors
        - essentially, this means `a * b + c` can be replaced by
          `fma(a, b, c)`
          - note that `-cl-mad-enable` allows `a * b + c ` to be replaced by
            `mad(a, b, c)`
- Chapter 7. OpenCL Numerical Compliance
  - 7.1. Rounding Modes
  - 7.2. INF, NaN and Denormalized Numbers
  - 7.3. Floating-Point Exceptions
  - 7.4. Relative Error as ULPs
  - 7.5. Edge Case Behavior
- Chapter 8. Image Addressing and Filtering
  - 8.1. Image Coordinates
  - 8.2. Addressing and Filter Modes
  - 8.3. Conversion Rules
  - 8.4. Selecting an Image From an Image Array

## Rounding

- OpenCL C
  - By default, conversions to integer type use the `_rtz` (round toward zero)
    rounding mode and conversions to floating-point type use the default
    rounding mode. The only default floating-point rounding mode supported is
    round to nearest even i.e the default rounding mode will be `_rte` for
    floating-point types.
  - `__ROUNDING_MODE__` and `cl_khr_select_fprounding_mode` are deprecated
  - The built-in math functions are not affected by the prevailing rounding
    mode in the calling environment, and always return the same value as they
    would if called with the round to nearest even rounding mode.
  - The built-in common functions are implemented using the round to nearest
    even rounding mode.
  - The built-in geometric functions are implemented using the round to
    nearest even rounding mode.
  - Round to nearest even is currently the only rounding mode required by the
    OpenCL specification for single precision and double precision operations
    and is therefore the default rounding mode. In addition, only static
    selection of rounding mode is supported. Dynamically reconfiguring the
    rounding modes as specified by the IEEE 754 spec is unsupported.
- OpenCL SPIR-V
  - For single precision operations, devices supporting the full profile must
    support Round to nearest even, therefore for full profile devices this is
    the default rounding mode for single precision operations. Devices
    supporting the embedded profile may support either Round to nearest even
    or Round toward zero as the default rounding mode for single precision
    operations.
  - Only static selection of rounding mode is supported. Dynamically
    reconfiguring the rounding mode as specified by the IEEE 754 spec is not
    supported.
  - Results of the following conversion instructions may include an optional
    FPRoundingMode decoration (all `Op*Convert*` ops)
  - If no rounding mode is specified explicitly via an FPRoundingMode
    decoration, then the default rounding mode for conversion operations is:
    - Round to nearest even, for conversions to floating-point types.
    - Round toward zero, for conversions from floating-point to integer types.
