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
