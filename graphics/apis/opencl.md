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

## Work-Groups

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

## Memory Model

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
