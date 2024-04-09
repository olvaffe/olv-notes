clvk
====

## Build

- steps
  - `git clone https://github.com/kpet/clvk.git`
  - `cd clvk`
  - `git submodule update --init --recursive`
  - `./external/clspv/utils/fetch_sources.py --deps llvm`
    - this helps `clspv` fetches `llvm`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DCLVK_CLSPV_ONLINE_COMPILER=ON -DCLVK_ENABLE_SPIRV_IL=OFF`
  - `ninja -C out`
  - `ninja -C out install` installs `libOpenCL.so.0.1`
    - no `clspv` because we enable online compiler
- compiler
  - apis
    - `clCreateProgramWithSource` accepts opencl c
    - `clCreateProgramWithBinary` accepts vendor-defined binary
      - clvk accepts llvm bitcode as `CL_PROGRAM_BINARY_TYPE_COMPILED_OBJECT`
        and accepts spirv as `CL_PROGRAM_BINARY_TYPE_EXECUTABLE`
    - `clCreateProgramWithIL` accepts spirv as intermediate language
  - clvk invokes `clspv` and `llvm-spirv` by default
    - `clspv -x=<FROM> --output-format=<TO>`
      - `FROM` can be `cl` (opencl c) or `ir` (llvm ir)
      - `TO` can be `spv` (spirv) or `bc` (llvm bitcode)
    - `llvm-spirv -r` reverse-translates spirv to llvm ir
    - this allows clvk to support all 3 of source/binary/il
- options
  - `-DCLVK_COMPILER_AVAILABLE=OFF` disables all but
    `clCreateProgramWithBinary` support
    - this option only needs `SPIRV-Tools` to work with spirv binary, and do
      not need `clspv` or `SPIRV-LLVM-Translator` to support source/il
  - `-DCLVK_ENABLE_SPIRV_IL=OFF` disables `clCreateProgramWithIL` support
    - this option removes need for `SPIRV-LLVM-Translator`
  - `-DCLVK_CLSPV_ONLINE_COMPILER=ON` links to static libraries from `clspv`
    and `SPIRV-LLVM-Translator`, removing the dependencies on `clspv` and
    `llvm-spirv` standalone tools
  - `-DCLVK_BUILD_TESTS=OFF` disables tests
- tests
  - `api_test`
  - `simple_test`
- packaging
  - `strip -g out/libOpenCL.so.1 && scp -C out/libOpenCL.so.1 <remote>:`

## OpenCL ICD Loaders

- `libOpenCL.so.1` is provided by
  - <https://github.com/KhronosGroup/OpenCL-ICD-Loader>, or more commonly,
  - <https://github.com/OCL-dev/ocl-icd>
- `OpenCL-ICD-Loader`
  - `OCL_ICD_ENABLE_TRACE=1` enables loader debug messages
  - `OCL_ICD_FILENAMES` specifies addtional ICDs that are `dlopen`ed before
    scanning the vendors path
  - `OCL_ICD_VENDORS` specifies an alternative vendors path rather than the
    default `/etc/OpenCL/vendors`
  - each `.icd` file under the vendors path contains a path that is `dlopen`ed
- `ocl-icd`
  - `OCL_ICD_DEBUG=15` enables loader debug messages
    - `1` for warnings
    - `2` for infos
    - `4` for function enter/leave
    - `8` dumps icd info
  - `OCL_ICD_VENDORS` specifies
    - the vendors path to be scanned,
    - the `.icd` file to be parsed, or
    - the icd library to be `dlopen`ed

## Rusticl

- `RUSTICL_ENABLE` must be specified to enable drivers
  - by default, no driver is enabled

## Build CTS

- steps
  - `git clone https://github.com/KhronosGroup/OpenCL-CTS.git`
  - `cd OpenCL-CTS`
  - `git clone https://github.com/KhronosGroup/OpenCL-Headers.git`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DCL_INCLUDE_DIR=OpenCL-Headers -DCL_LIB_DIR=../clvk/out -DOPENCL_LIBRARIES=OpenCL`
    - for simplicity, we don't use `OpenCL-ICD-Loader`
  - `ninja -C out`
    - this builds various test executables from `test_conformance/*`
    - `find out/test_conformance -type f -executable`
- options
  - `-DCL_INCLUDE_DIR=OpenCL-Headers`
    - it's used in `include_directories(SYSTEM ${CL_INCLUDE_DIR})` which adds
      `-I`
  - `-DCL_LIB_DIR=../clvk/out`
    - it's used in `link_directories(${CL_LIB_DIR})` which adds `-L` and
      `-Wl,-rpath`
  - `-DOPENCL_LIBRARIES=OpenCL`
    - it's added to `TARGET_LINK_LIBRARIES`
- packaging
  - `oclctsdir=ocl-cts-$(date +%Y%m%d)`
  - `mkdir -p $oclctsdir`
  - `find out/test_conformance -type f -executable -exec ln -sf ../{} $oclctsdir/ \;`
  - `strip -g $oclctsdir/*`
  - `tar --zstd -hcf $oclctsdir.tar.zst $oclctsdir`

## clpeak

- `device_info_t`
  - `maxAllocSize` is `CL_DEVICE_MAX_MEM_ALLOC_SIZE` (e.g., 4GB)
  - `maxGlobalSize` is `CL_DEVICE_GLOBAL_MEM_SIZE` (e.g., 32GB)
  - `maxWGSize` is `CL_DEVICE_MAX_WORK_ITEM_SIZES[0]` (e.g., 512)
  - `numCUs` is `CL_DEVICE_MAX_COMPUTE_UNITS` (e.g., 96)
  - `globalBWMaxSize` is hardcoded to 512MB
  - `computeWgsPerCU` is hardcoded to 2048
- `clPeak::runGlobalBandwidthTest`
  - `global_bandwidth_v1_local_offset` kernel
    - it sums `FETCH_PER_WI` values, each `get_local_size` apart, and writes
      the result
  - `global_bandwidth_v1_global_offset` kernel
    - it sums `FETCH_PER_WI` values, each `get_global_size` apart, and writes
      the result
    - that's bogus
- `clPeak::runComputeSP`
  - `compute_sp_v1` kernel
    - it computes `mad` with `get_local_id` for 1024 times and writes the
      result
  - `globalSize` is scaled by `numCUs` and `maxWGSize`
- `clPeak::runComputeInteger`
  - `compute_integer_v1` kernel
    - it computes `a*b+c` with `get_local_id` for 1024 times and writes the
      result
  - `globalSize` is scaled by `numCUs` and `maxWGSize`

## CTS: `test_conversions`

- `InitCL`, among other things, queries and prints device info
  - `CL_DEVICE_PROFILE` can be `FULL_PROFILE` or `EMBEDDED_PROFILE`
    - if `EMBEDDED_PROFILE`, int64 support is optional and depends on
      `cles_khr_int64` extension
  - `CL_DEVICE_EXTENSIONS` contains all extensions
    - check for `cl_khr_fp64` for `double` support
  - `CL_DEVICE_SINGLE_FP_CONFIG` (and `CL_DEVICE_DOUBLE_FP_CONFIG` and
    `CL_DEVICE_HALF_FP_CONFIG`)
    - `CL_FP_DENORM` supports denorms
    - `CL_FP_INF_NAN` supports inf/nan
    - `CL_FP_ROUND_TO_NEAREST` supports RTE
    - `CL_FP_ROUND_TO_ZERO` supports RTZ
    - `CL_FP_ROUND_TO_INF` supports RTP and RTN (positive and negative
      infinity)
    - `CL_FP_FMA` supports FMA
    - `CL_FP_CORRECTLY_ROUNDED_DIVIDE_SQRT` supports correct rounding for
      divide and sqrt
    - `CL_FP_SOFT_FLOAT` uses software impl
  - clvk
    - incorrectly assumes `VkPhysicalDeviceFeatures::shaderInt64`
    - if `VkPhysicalDeviceFeatures::shaderFloat64`, advertises `cl_khr_fp64`
    - correctly assumes `CL_FP_ROUND_TO_NEAREST` and never advertises
      `CL_FP_ROUND_TO_ZERO`
- `MakeProgram`
  - for `float_long` test case, the kernel is something like

    __kernel void test_implicit_float_long( __global long *src, __global float *dest )
    {
       size_t i = get_global_id(0);
       dest[i] =  src[i]; // or with explicit convert_float
    }
- `InitData` initializes input data
  - for `float_long` test case,
    - `init_long` generates various bit patterns
- `PrepareReference` calculates the reference values
  - for `float_long` test case,
    - `set_round` calls `fesetround(FE_TONEAREST)`
    - `long2float_many` simply casts `cl_long` to `double`

## CTS: `test_cl_copy_images`

- `test_cl_copy_images --help`
  - specify test names to limit tests
  - undocumented, but specify a channel type such as `CL_UNORM_INT8` to limit
    to a channel type
  - undocumented, but specify a channel order such as `CL_R` to limit
    to a channel order
- `test_image_set` is called with varous `MethodsToTest` (e.g., 1D, 2D, etc.)
  - `imageType` is set to the destination image type such as
    `CL_MEM_OBJECT_IMAGE3D`
  - `get_format_list` returns all supported `cl_image_format`
  - `filter_formats` limits to the specified channel type/order, if specified
- `test_copy_image_set_3D_2D_array` selects random sizes for src and dst
  images and calls `test_copy_image_size_3D_2D_array`
- `test_copy_image_size_3D_2D_array` selects random regions and calls
  `test_copy_image_generic`
- `test_copy_image_generic`
  - it generates random data and calls `create_image` to create/initialize
    src/dst images
  - `clEnqueueCopyImage` copies
  - `copy_image_data` simulates the copy
  - verification
- for `2Darrayto3D,CL_R,CL_UNORM_INT8`, clvk translates it to
  - src image
    - `vkCreateImage` a 329x349x53, tiled, `VK_FORMAT_R8_UNORM` 2D array image
    - `vkCreateImageView` two idential image views for the entire image
    - `vkCreateBuffer` a buffer of size 6085513 (329x349x53)
    - create and submit 2 command buffers
      - they transition the image from undefined to general, and
        `vkCmdCopyImageToBuffer` the whole image to the buffer
      - they are submitted in reverse order
    - create and submit 1 command buffers
      - it `vkCmdCopyBufferToImage` from the whole buffer to the image
  - dst image
    - `vkCreateImage` a 58x48x58, tiled, `VK_FORMAT_R8_UNORM` 3D image
    - `vkCreateImageView` two idential image views for the entire image
    - `vkCreateBuffer` a buffer of size 161472 (58x48x58)
    - create and submit 2 command buffers
      - they transition the image from undefined to general, and
        `vkCmdCopyImageToBuffer` the whole image to the buffer
      - they are submitted in reverse order
    - create and submit 1 command buffers
      - it `vkCmdCopyBufferToImage` from the whole buffer to the image
  - copy
    - create and submit 1 command buffers
      - it `vkCmdCopyImage` 58x48 pixels from layer 28 of the src image to
        depth 49 of the dst image
    - `vkCreateBuffer` a buffer of size 161472 (58x48x58)
    - create and submit 1 command buffers
      - it `vkCmdCopyImageToBuffer` the whole dst image to a buffer for
        verification
