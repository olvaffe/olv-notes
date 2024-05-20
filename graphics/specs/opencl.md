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
    - `CL_DEVICE_SINGLE_FP_CONFIG` describes fp32 caps
      - only `CL_FP_ROUND_TO_NEAREST` and `CL_FP_INF_NAN` are required
    - `CL_DEVICE_DOUBLE_FP_CONFIG` describes fp64 caps
      - optional, but if supported, these are required
        - `CL_FP_FMA | CL_FP_ROUND_TO_NEAREST | CL_FP_INF_NAN | CL_FP_DENORM`
    - `CL_DEVICE_MAX_CONSTANT_BUFFER_SIZE` at least 64KB (full) or 1KB
      (embedded)
    - `CL_DEVICE_LOCAL_MEM_SIZE` at least 32KB (full) or 1KB (embedded)
    - `CL_DEVICE_MAX_NUM_SUB_GROUPS`
      - max queued waves
    - `CL_DEVICE_SUB_GROUP_INDEPENDENT_FORWARD_PROGRESS`
      - true
    - `CL_DEVICE_PREFERRED_WORK_GROUP_SIZE_MULTIPLE`
      - minimum wave size?
    - `CL_DEVICE_HALF_FP_CONFIG` describes fp16 caps
      - optional, but if supported, either of these is required
        - `CL_FP_ROUND_TO_ZERO` or
        - `CL_FP_ROUND_TO_NEAREST | CL_FP_INF_NAN`
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
    - optional features
      - `__opencl_c_3d_image_writes`
      - `__opencl_c_atomic_order_acq_rel`
      - `__opencl_c_atomic_order_seq_cst`
      - `__opencl_c_atomic_scope_device`
      - `__opencl_c_atomic_scope_all_devices`
      - `__opencl_c_device_enqueue`
      - `__opencl_c_generic_address_space`
      - `__opencl_c_fp64`
      - `__opencl_c_images`
      - `__opencl_c_int64`
      - `__opencl_c_pipes`
      - `__opencl_c_program_scope_global_variables`
      - `__opencl_c_read_write_images`
      - `__opencl_c_subgroups`
      - `__opencl_c_work_group_collective_functions`
      - `__opencl_c_integer_dot_product_input_4x8bit_packed`
      - `__opencl_c_integer_dot_product_input_4x8bit`
      - `__opencl_c_kernel_clock_scope_device`
      - `__opencl_c_kernel_clock_scope_work_group`
      - `__opencl_c_kernel_clock_scope_sub_group`
    - extensions
  - 6.3. Supported Data Types
    - Built-in Scalar Data Types
      - `bool`
      - `char`, `uchar`
      - `short`, `ushort`
      - `int`, `uint`
      - `long`, `ulong`
      - `float`, `double` (requires `__opencl_c_fp64`), `half`
      - `size_t`, `ptrdiff_t`, `intptr_t`, `uintptr_t`
      - `void`
    - without `cl_khr_fp16`, `half` is limited
      - it can only be used to declare a pointer to a buffer that contains
        `half` values
    - Built-in Vector Data Types
      - `N` can be 2, 3, 4, 8, 16
      - `charN`, `ucharN`
      - `shortN`, `ushortN`
      - `intN`, `uintN`
      - `longN`, `ulongN`
      - `floatN`, `doubleN` (requires `__opencl_c_fp64`), `halfN` (requires
        `cl_khr_fp16`)
    - Other Built-in Data Types
      - `image1d_t`, `image1d_buffer_t`, `image1d_array_t`
      - 8 variants of `image2d_t` 
        - all combinations of `array`, `depth`, and `msaa`
      - `image3d_t`
      - `sampler_t`
      - `queue_t`
      - `ndrange_t`
      - `clk_event_t`
      - `reserve_id_t`
      - `event_t`
      - `cl_mem_fence_flags`
    - Alignment of Types
      - A data item in memory is always aligned to the size of the data type
        in bytes. For example, a `float4` variable will be aligned to a
        16-byte boundary, a `char2` variable will be aligned to a 2-byte
        boundary.
      - note that the size of 3-component vector data types are the same as
        4-component vector data types
      - A built-in data type that is not a power of two bytes in size must be
        aligned to the next larger power of two. This rule applies to built-in
        types only, not structs or unions.
  - 6.4. Conversions and Type Casting
    - implicit conversions and c-style explicit casts
      - between scalar built-in types
      - from scalar to vector
      - disallowed between built-in vector types
    - explicit conversions
      - `convert_destType(sourceType)`
      - `convert_destType<_sat><_roundingMode>(sourceType)`
        - if rounding mode is not specified,
          - `_rtz` is assumed for conversion to integer types
          - the default rounding mode is assumed for conversion to float types
            - the default rounding mode for fp32 and fp64 is `_rte`
            - the default rounding mode for fp16 is `_rte` if supported;
              `_rtz` otherwise
      - the vector size must match
    - reinterpreting types
      - the `union` trick
      - `as_destType(sourceType)`
  - 6.5. Operators
  - 6.6. Vector Operations
    - vector operations are component-wise
  - 6.7. Address Space Qualifiers
    - `__global` is supported by
      - program scope variables (if `__opencl_c_program_scope_global_variables`)
      - `static` or `extern` local variables
      - pointers
    - `__private` is supported by
      - local scope variables
      - function arguments and return types
      - pointers
    - `__constant` is supported by
      - program scope variables
      - kernel scope variables
      - string literals
      - pointers
    - `__local` is supported by
      - kernel scope variables
      - pointers
    - kernel function arguments that are pointers or arrays must have
      `__global`, `__local` or `__constant`.
  - 6.8. Access Qualifiers
    - `__read_only` (default), `__write_only`, `__read_write`
  - 6.9. Function Qualifiers
    - `__kernel`
    - `__attribute__((vec_type_hint(float4)))` means most operations in the
      kernel function are explicitly vectorized using `float4`
    - `__attribute__((work_group_size_hint(X, Y, Z)))` means `local_work_size`
      is most likely `(X, Y, Z)`
  - 6.10. Storage-Class Specifiers
    - `typedef`, `extern`, `static`
  - 6.11. Restrictions
    - recursion is not supported
    - the return type of a kernel function must be void
    - `half` is not supported as `half` can be used as a storage format only
      and is not a data type on which floating-point arithmetic can be
      performed
      - unless `cl_khr_fp16`
    - a function cannot be called main
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
      - always RTE
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
    - integer functions
      - `abs`, `mad24`, etc.
    - common functions
      - `clamp`, `max`, `min`, etc.
    - geometric functions
      - `cross`, `dot`, `distance`, etc.
    - relational functions
      - `isequal`, `isgreater`, etc.
    - vector data load and store functions
      - `vloadn`, `vstoren`, etc.
    - synchronization functions
      - `work_group_barrier`
      - `sub_group_barrier`
    - legacy explicit memory fence functions
    - address space qualifier functions
      - `to_global`, `to_local`, etc.
    - async copies
      - `async_work_group_copy`, `prefetch`, etc.
    - atomic functions
      - `atomic_load`, `atomic_store`, `atomic_work_item_fence`, etc.
    - miscellaneous vector functions
      - `vec_step`, `shuffle`
    - printf
      - `printf`
    - image read and write functions
      - `read_image{f,h,i,ui}`
      - `write_image{f,h,i,ui}`
      - `get_image_{width,height,channel_data_type,channel_order,dim,array_size,num_samples,num_mip_levels}`
    - work-group collective functions
      - `work_group_all`, `work_group_any`, `work_group_broadcast`, etc.
    - work-group collective uniform arithmetic functions
      - `work_group_reduce_logical_and`, etc.
    - pipe functions
      - `read_pipe`, `write_pipe`, etc.
    - enqueuing kernels
      - `enqueue_kernel`, etc.
    - sub-group functions
      - `sub_group_all`, `sub_group_any`, `sub_group_broadcast`, etc.
    - kernel clock functions
      - `clock_read_device`, `clock_read_work_group`, `clock_read_sub_group`,
        etc.
- Chapter 7. OpenCL Numerical Compliance
  - 7.1. Rounding Modes
    - RTE is currently the only rounding mode required by the OpenCL
      specification for single precision and double-precision operations and
      is therefore the default rounding mode
    - If the `cl_khr_fp16` extension macro is supported, then if
      `CL_FP_ROUND_TO_NEAREST` is supported, the default rounding mode for
      half-precision floating-point operations will be round to nearest even;
      otherwise the default rounding mode will be round to zero.
    - conversion to float types defaults to default rounding mode
    - conversion to integer tyeps defaults to RTZ
    - math functions, common functions, and geometric Functions are
      implemented with RTE
    - `__ROUNDING_MODE__` and `cl_khr_select_fprounding_mode` are deprecated
  - 7.2. INF, NaN and Denormalized Numbers
  - 7.3. Floating-Point Exceptions
  - 7.4. Relative Error as ULPs
    - add, sub, mul, fma, and conversions must be correctly rounded
    - ULP, unit in the last place
      - if `x` lies between two finite consecutive numbers `(a, b)`, then
        `ulp(x)= |b - a|`
      - correctly rounding rounds `x` to either `a` or `b` depending on the
        values and the rounding modes
      - RTE has `<=0.5ulp`
      - RTZ has `<1ulp`
  - 7.5. Edge Case Behavior
- Chapter 8. Image Addressing and Filtering
  - 8.1. Image Coordinates
  - 8.2. Addressing and Filter Modes
  - 8.3. Conversion Rules
    - unorm/snorm to fp32/fp16 conversion has precision `<=1.5ulp`
    - fp32/fp16 to unorm/snorm is suggested to use `_rte`
    - fp16 to fp32 is lossless
    - fp32 to fp16 can use `_rte` or `_rtz`
  - 8.4. Selecting an Image From an Image Array
