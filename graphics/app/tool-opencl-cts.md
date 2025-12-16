OpenCL CTS
==========

## Build

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
