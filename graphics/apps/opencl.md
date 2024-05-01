OpenCL Apps
===========

## Loaders

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
  - `OPENCL_LAYER_PATH` specifies an alternative layers path rather than the
    default `/etc/OpenCL/layers`
    - each `.lay` file under the layers path contains a path that is `dlopen`ed
    - all layers are implicitly loaded?
  - `OPENCL_LAYERS` specifies additional layers that are `dlopen`ed after
    scanning the layers path
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

## Layers

- Intercept Layer for OpenCL
  - build
    - `git clone https://github.com/intel/opencl-intercept-layer.git`
    - `cmake -S . -B out -G Ninja`
  - config `~/clintercept.conf`
    - env `CLI_<option>` sets the same option
    - Setup and Loading Controls
      - `OpenCLFileName` (string)
        - the path of the real icd loader
    - Logging Controls
      - `LogToFile` (bool)
        - log to `clintercept_log.txt` instead of stderr
      - `CallLogging` (bool)
        - log all cl calls
      - `CallLoggingThreadId` or `CallLoggingThreadNumber` (bool)
        - log thread id or number
      - `CallLoggingElapsedTime` (bool)
        - log call time
      - `ErrorLogging` (bool)
        - log cl errors
      - `FlushFiles` (bool)
        - flush after each log write
      - `DumpDir` (string)
        - default to `~/CLIntercept_Dump/<Process Name>`
    - Reporting Controls
    - Controls for Dumping and Injecting Programs and Build Options
      - `DumpProgramSource` (bool)
        - dump all program source
      - `DumpProgramBinaries` (bool)
        - dump all program binaries
      - `AppendBuildOptions` (string)
        - append build options
      - `AppendLinkOptions` (string)
        - append link options
    - Controls for Emulating Features
    - Controls for Automatically Creating SPIR-V Modules
    - Controls for Dumping Buffers and Images
    - Device Partitioning Controls
    - Capture and Replay Controls
    - AubCapture Controls
    - Execution Controls
    - Platform and Device Query Overrides
    - Precompiled Kernel and Builtin Kernel Override Controls
- OpenCL Kernel Profiler
  - build
    - `git clone https://github.com/rjodinchr/opencl-kernel-profiler.git`
    - `cmake -S . -B out -G Ninja -DPERFETTO_SDK_PATH=~/projects/perfetto/sdk -DOPENCL_HEADER_PATH=/usr/include`
  - use
    - `OPENCL_LAYERS=<path>/libopencl-kernel-profiler.so CLKP_TRACE_MAX_SIZE=100000 prog` will generate
      `opencl-kernel-profiler.trace`
      - without `CLKP_TRACE_MAX_SIZE`, the trace file is truncated after 1MB

## Tools

- <https://github.com/KhronosGroup/OpenCL-Guide/blob/main/chapters/os_tooling.md>
- OpenCL C to LLVM IR
  - `clang -c -target spir -O0 -emit-llvm -o test.bc test.cl`
  - `-x` specifies the language explicitly
    - `c` for c
    - `c++` for c++
    - `cl` for opencl
    - `cuda` for cuda
    - otherwise, it is based on the filename suffix
  - `-std` specifies the language standard
    - `c17` for c17
    - `c++17` for c++17
    - `cl3.0` for opencl c 3.0
    - `cuda` for cuda
  - `-target` specifies the target arch
    - `x86_64-linux-gnu` for x86-64
    - `aarch64-linux-gnu` for aarch64
    - `spirv64-unknown-unknown` for spirv 64
      - it was called `spir64-unknown-unknown`
      - it invokes `llvm-spirv` to convert from llvm ir to spirv
  - `-emit-llvm` generates llvm ir instead
    - it cannot be used with linking so `-c` must also be specified
  - `-g` generates debug info
- LLVM IR to SPIR-V
  - `llvm-spirv test.bc -o test.spv`
- SPIR-V
  - `clCreateProgramWithIL` can load `test.spv`
  - the spirv generated this way is not compatible with vulkan
    - it uses capabilities that vulkan drivers do not support, such as
      - `OpCapability Addresses`
      - `OpCapability Linkage`
      - `OpCapability Kernel`
    - use `clspv` for that purpose instead
  - `spirv-dis test.spv` to disassemble
- SPIR-V to NIR
  - `spirv2nir test.spv`
    - must not include debug info
      - `Unsupported extension: OpenCL.DebugInfo.100`
    - must target `spirv32` rather than `spirv64`
  - `--optimize` to run basic optimizations

## Rusticl

- `RUSTICL_ENABLE` must be specified to enable drivers
  - by default, no driver is enabled

## clvk

- build
  - `git clone https://github.com/kpet/clvk.git`
  - `cd clvk`
  - `git submodule update --init --recursive`
  - `./external/clspv/utils/fetch_sources.py --deps llvm`
    - this helps `clspv` fetches `llvm`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DCLVK_CLSPV_ONLINE_COMPILER=ON -DCLVK_ENABLE_SPIRV_IL=OFF`
    - `-DCLVK_PERFETTO_ENABLE=ON -DCLVK_PERFETTO_SDK_DIR=<path-to-perfetto-sdk> -DCLVK_PERFETTO_BACKEND=System`
    - `-DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `ninja -C out`
    - no `clspv` because we enable online compiler
  - `strip -g out/libOpenCL.so.1 && scp -C out/libOpenCL.so.1 <remote>:`
- build options
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
- env
  - env `CLVK_<OPT_IN_UPPERCASE>` sets option `<opt_in_lowercase>` in
    `src/config.def`
  - `CLVK_LOG=4` to log everything
    - from 4 to 0: debug, info, warn, error, fatal
  - `CLVK_CLSPV_OPTIONS=foo` to pass extra options to clspv
    - options from `clBuildProgram`, clvk-generated, and this env var are
      combined
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
- tests
  - `api_test`
  - `simple_test`

## clspv

- clspv
  - <https://github.com/google/clspv>
  - depends on clang, llvm, SPIRV-Headers, and SPIRV-Tools
  - provides tool (`clspv`) and library (`libclspv_core`) to convert opencl c
    to vk-ready spirv
- build
  - `git clone https://github.com/google/clspv`
  - `python3 utils/fetch_sources.py`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DCLSPV_SHARED_LIB=ON`
    - `-DCMAKE_C_VISIBILITY_PRESET=hidden -DCMAKE_CXX_VISIBILITY_PRESET=hidden`
      - when `-DCLSPV_SHARED_LIB=ON`, it is desirable to hide llvm symbols
    - `-DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `ninja -C out clspv_core`
- the only entrypoint is `clspvCompileFromSourcesString`
  - compiler options
    - `-cl-std=CL3.0` selects CLC std
    - `-inline-entry-points` inlines all entrypoints
      - required for `__opencl_c_generic_address_space` feature
    - `-cl-single-precision-constant` treats double-precision float literals
      as single-precision
    - `-cl-kernel-arg-info` produces kernel argument info
    - `-rounding-mode-rte=16,32,64` sets `RoundingModeRTE`
    - `-rewrite-packed-structs` rewrites packed structs as i8 array
      - vk must support `storageBuffer8BitAccess`
    - `-std430-ubo-layout` uses more packed std430 layout
      - vk must support `uniformBufferStandardLayout`
    - `-decorate-nonuniform` decorates `NonUniform`
      - vk must support non-uniform descriptor indexing
    - `-hack-convert-to-float` adds a workaround for radv/aco
    - `-arch=spir` selects 32-bit pointers
    - `-spv-version=1.5` targets spirv 1.5
    - `-max-pushconstant-size=128` sets push contant size limit
      - it should match vk `maxPushConstantsSize`
    - `-max-ubo-size=16384`
      - it should match vk `maxUniformBufferRange`
    - `-global-offset` enables support for global offset (`get_global_offset`?)
    - `-long-vector` enables support for vectors of width 8 and 16
    - `-module-constants-in-storage-buffer` collects module-socped
      `__constants` into an ssbo
    - `-cl-arm-non-uniform-work-group-size` enables
      `cl_arm_non_uniform_work_group_size` extension
- `clspvCompileFromSourcesString` internal
  - `ProgramToModule` uses clang to translate CLC to `llvm::Module`
  - `CompileModule` compiles the module to spirv
    - `LinkBuiltinLibrary` links in the built-in library
      - `clspv--.bc` and `clspv64--.bc` are from libclc
      - they are parsed into an `llvm::Module` and linked into the source
        module
    - `RunPassPipeline` runs the passes to produce spirv
      - `clspv::RegisterClspvPasses` registers clspv passes to llvm
      - a `llvm::PassBuilder` is used to build the pass pipeline
      - the default optimization level is `llvm::OptimizationLevel::O2`
      - `registerPipelineStartEPCallback` adds passes to the start of
        the default pipeline
      - `registerOptimizerLastEPCallback` adds passes to the end of the
        default pipeline
        - `clspv::SPIRVProducerPass` is the last pass that generates spirv
      - `clspv::SPIRVProducerPass::run` generates spirv
        - `outputHeader` generates the header
        - `GenerateSamplers` generates samplers
        - `GenerateGlobalVar` generates global variables
        - `GenerateResourceVars` generates resource variables
        - `GenerateWorkgroupVars` generates workgroup variables
        - `GenerateFuncPrologue`, `GenerateFuncBody`, and
          `GenerateFuncEpilogue` generate functions
        - `GenerateModuleInfo` generates module info
        - `GenerateReflection` generates reflection
        - `WriteSPIRVBinary` generates the binary
- llvm supports `-print-after-all` to print IRs after each pass; it looks like
  clspv applies these passes in order
  - llvm early passes
    - `AlwaysInlinerPass`
    - `CoroConditionalWrapper`
    - `AnnotationRemarksPass`
    - `Annotation2MetadataPass`
    - `ForceFunctionAttrsPass`
  - `registerPipelineStartEPCallback`
    - `clspv::AnnotationToMetadataPass`
    - `clspv::WrapKernelPass`
    - `clspv::NativeMathPass`
    - `clspv::ZeroInitializeAllocasPass`
    - `clspv::KernelArgNamesToMetadataPass`
    - `clspv::AddFunctionAttributesPass`
    - `clspv::AutoPodArgsPass`
    - `clspv::DeclarePushConstantsPass`
    - `clspv::DefineOpenCLWorkItemBuiltinsPass`
    - `clspv::OpenCLInlinerPass`
    - `clspv::UndoByvalPass`
    - `clspv::UndoSRetPass`
    - `InferAddressSpacesPass`
    - `PromotePass`
    - `clspv::ClusterPodKernelArgumentsPass`
    - `clspv::InlineEntryPointsPass`
    - `clspv::FunctionInternalizerPass`
    - `clspv::ReplaceOpenCLBuiltinPass`
    - `clspv::FixupBuiltinsPass`
    - `clspv::ThreeElementVectorLoweringPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::ReplacePointerBitcastPass`
    - `DCEPass`
    - `clspv::HideConstantLoadsPass`
    - `InstCombinePass`
    - `clspv::InlineFuncWithImageMetadataGetterPass`
    - `clspv::InlineFuncWithPointerBitCastArgPass`
    - `clspv::InlineFuncWithPointerToFunctionArgPass`
    - `clspv::InlineFuncWithSingleCallSitePass`
    - `clspv::InlineFuncWithReadImage3DNonLiteralSamplerPass`
    - `PromotePass`
    - `clspv::LogicalPointerToIntPass`
    - `PromotePass`
    - `SROAPass`
    - `InstCombinePass`
    - `InferAddressSpacesPass`
  - llvm default passes
    - `InferFunctionAttrsPass`
    - `CoroEarlyPass`
    - `LowerExpectIntrinsicPass`
    - `SimplifyCFGPass`
    - `SROAPass`
    - `EarlyCSEPass`
    - `OpenMPOptPass`
    - `IPSCCPPass`
    - `CalledValuePropagationPass`
    - `GlobalOptPass`
    - `PromotePass`
    - `InstCombinePass`
    - `SimplifyCFGPass`
    - `AlwaysInlinerPass`
    - `RequireAnalysisPass<GlobalsAA, Module>`
    - `InvalidateAnalysisPass<AAManager>`
    - `RequireAnalysisPass<ProfileSummaryAnalysis, Module>`
    - `InlinerPass`
    - `PostOrderFunctionAttrsPass`
    - `OpenMPOptCGSCCPass`
    - `SROAPass`
    - `EarlyCSEPass`
    - `SpeculativeExecutionPass`
    - `JumpThreadingPass`
    - `CorrelatedValuePropagationPass`
    - `SimplifyCFGPass`
    - `InstCombinePass`
    - `AggressiveInstCombinePass`
    - `LibCallsShrinkWrapPass`
    - `TailCallElimPass`
    - `SimplifyCFGPass`
    - `ReassociatePass`
    - `ConstraintEliminationPass`
    - `LoopSimplifyPass`
    - `LCSSAPass`
    - `LoopInstSimplifyPass`
    - `LoopSimplifyCFGPass`
    - `LICMPass`
    - `LoopRotatePass`
    - `LICMPass`
    - `SimpleLoopUnswitchPass`
    - `SimplifyCFGPass`
    - `InstCombinePass`
    - `LoopSimplifyPass`
    - `LCSSAPass`
    - `LoopIdiomRecognizePass`
    - `IndVarSimplifyPass`
    - `LoopDeletionPass`
    - `LoopFullUnrollPass`
    - `SROAPass`
    - `VectorCombinePass`
    - `MergedLoadStoreMotionPass`
    - `GVNPass`
    - `SCCPPass`
    - `BDCEPass`
    - `InstCombinePass`
    - `JumpThreadingPass`
    - `CorrelatedValuePropagationPass`
    - `ADCEPass`
    - `MemCpyOptPass`
    - `DSEPass`
    - `MoveAutoInitPass`
    - `LoopSimplifyPass`
    - `LCSSAPass`
    - `LICMPass`
    - `CoroElidePass`
    - `SimplifyCFGPass`
    - `InstCombinePass`
    - `PostOrderFunctionAttrsPass`
    - `RequireAnalysisPass<ShouldNotRunFunctionPassesAnalysis, Function>`
    - `CoroSplitPass`
    - `InvalidateAnalysisPass<ShouldNotRunFunctionPassesAnalysis>`
    - `DeadArgumentEliminationPass`
    - `CoroCleanupPass`
    - `GlobalOptPass`
    - `GlobalDCEPass`
    - `EliminateAvailableExternallyPass`
    - `ReversePostOrderFunctionAttrsPass`
    - `RecomputeGlobalsAAPass`
    - `Float2IntPass`
    - `LowerConstantIntrinsicsPass`
    - `LoopSimplifyPass`
    - `LCSSAPass`
    - `LoopRotatePass`
    - `LoopDeletionPass`
    - `LoopDistributePass`
    - `InjectTLIMappings`
    - `LoopVectorizePass`
    - `InferAlignmentPass`
    - `LoopLoadEliminationPass`
    - `InstCombinePass`
    - `SimplifyCFGPass`
    - `VectorCombinePass`
    - `InstCombinePass`
    - `LoopUnrollPass`
    - `WarnMissedTransformationsPass`
    - `SROAPass`
    - `InferAlignmentPass`
    - `InstCombinePass`
    - `LoopSimplifyPass`
    - `LCSSAPass`
    - `LICMPass`
    - `AlignmentFromAssumptionsPass`
    - `LoopSinkPass`
    - `InstSimplifyPass`
    - `DivRemPairsPass`
    - `TailCallElimPass`
    - `SimplifyCFGPass`
  - `registerOptimizerLastEPCallback`
    - `clspv::StripFreezePass`
    - `clspv::UnhideConstantLoadsPass`
    - `clspv::UndoInstCombinePass`
    - `clspv::FunctionInternalizerPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::ReplaceLLVMIntrinsicsPass`
    - `DCEPass`
    - `clspv::UndoBoolPass`
    - `clspv::UndoTruncateToOddIntegerPass`
    - `LowerSwitchPass`
    - `StructurizeCFGPass`
    - `clspv::FixupStructuredCFGPass`
    - `clspv::ReorderBasicBlocksPass`
    - `clspv::UndoGetElementPtrConstantExprPass`
    - `clspv::SplatArgPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::ReplacePointerBitcastPass`
    - `DCEPass`
    - `clspv::UndoTranslateSamplerFoldPass`
    - `clspv::ShareModuleScopeVariablesPass`
    - `clspv::SpecializeImageTypesPass`
    - `clspv::AllocateDescriptorsPass`
    - `clspv::DirectResourceAccessPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::RemoveUnusedArguments`
    - `DCEPass`
    - `clspv::SplatSelectConditionPass`
    - `clspv::SignedCompareFixupPass`
    - `clspv::ScalarizePass`
    - `clspv::RewriteInsertsPass`
    - `clspv::LowerPrivatePointerPHIPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::ReplacePointerBitcastPass`
    - `clspv::SimplifyPointerBitcastPass`
    - `clspv::UBOTypeTransformPass`
    - `clspv::SetImageMetadataPass`
    - `clspv::SPIRVProducerPass`
  - llvm late passes
    - `GlobalDCEPass`
    - `ConstantMergePass`
    - `CGProfilePass`
    - `RelLookupTableConverterPass`
    - `AnnotationRemarksPass`

## CTS

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
