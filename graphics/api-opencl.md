OpenCL
======

# References

- http://www.khronos.org/registry/cl/specs/opencl-1.0.48.pdf (obvious :) ),
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
