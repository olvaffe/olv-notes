clvk
====

## clvk

- build
  - `git clone https://github.com/kpet/clvk.git`
  - `cd clvk`
  - `git submodule update --init --recursive`
  - `./external/clspv/utils/fetch_sources.py --deps llvm`
    - this helps `clspv` fetches `llvm`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Release -DCLVK_CLSPV_ONLINE_COMPILER=ON -DCLVK_ENABLE_SPIRV_IL=OFF`
    - `-DCLVK_PERFETTO_ENABLE=ON -DCLVK_PERFETTO_SDK_DIR=<path-to-perfetto-sdk> -DCLVK_PERFETTO_BACKEND=System`
    - `-DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `ninja -C out OpenCL`
    - no `clspv` because we enable online compiler
  - `strip -g out/libOpenCL.so.0.1 && scp -C out/libOpenCL.so.0.1 <remote>:`, or
  - `clvkdir=clvk-$(date +%Y%m%d) && tar -zcf $clvkdir.tar.gz --transform="s,^,$clvkdir/," -C out libOpenCL.so.0.1`
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
- `cvk_event` is pure sw
  - it tracks the progress of a `cvk_command`, which is a `clEnqueue*` call
  - `m_status` progresses over several states
    - `CL_QUEUED` is the initial status, meaning the cmd has been enqueued
    - `CL_SUBMITTED`
    - `CL_RUNNING`
    - `CL_COMPLETE`
  - `cvk_event::set_status` sets the status
    - if profiling, `set_profiling_info_from_monotonic_clock` is called to
      record the timestamp for the new status
    - if it reaches `CL_COMPLETE`, all registered callbacks are executed and
      `m_cv.notify_all` is called
  - `cvk_event::wait` blocks the calling thread until `m_status` reaches
    `CL_OMPLETE` or error
  - each `cvk_command` (a `clEnqueue` call) owns a `cvk_event`
    - the event status is `CL_QUEUED` initially
    - when `cvk_command_queue` flushes, the event status is set to
      `CL_SUBMITTED` just before the cmd is moved to `cvk_executor_thread`
    - when the executor calls `cvk_command::execute`,
      - the event status is set to `CL_RUNNING`
      - `do_action` is called and returns a status, normally `CL_COMPLETE`
      - the event status is set to the returned status
- `clEnqueue*` enqueues a `cvk_command` to a `cvk_command_queue`
  - it creates a proper subclass of `cvk_command` to encapsulate all args
  - it calls `cvk_command_queue::enqueue_command_with_deps` to enqueue the cmd
    - cmds on the same cmdq will be executed in order
  - `cvk_command::set_dependencies` sets all input events as dependencies
    - when `cvk_command::execute` is called later to execute the cmd, all
      dependencies will be waited before cmd execution
  - `cvk_command_queue::enqueue_command` adds the cmd to
    - the current `cvk_command_batch`, if the cmd is batchable, or
    - the current `cvk_command_group`, if the cmd is not batchable
  - if `blocking` is true, `clEnqueue*` blocks until the cmd is completed
    - it calls `wait_for_events` on the event associated with the cmd
    - this flushes the queue and waits for the event to become completed
- `clEnqueueWriteBuffer` creates a `cvk_command` to remember the args and
  calls `cvk_command_queue::enqueue_command_with_deps`
  - the args are saved in a `cvk_command_buffer_host_copy`, a subclass of
    `cvk_command`, with type `CL_COMMAND_WRITE_BUFFER`
  - because `cvk_command_buffer_host_copy` is not batchable,
    - `cvk_command_queue::end_current_command_batch` ends and enqueues the
      current batch, if any
    - `cvk_command_queue::enqueue_command` enqueues this write buffer cmd,
      which adds it to `m_groups.back()->commands`
- `clEnqueueNDRangeKernel` creates a `cvk_command` to remember the args and
  calls `cvk_command_queue::enqueue_command_with_deps`
  - the args are saved in a `cvk_command_kernel`, a subclass of `cvk_command`,
    with type `CL_COMMAND_NDRANGE_KERNEL`
  - because `cvk_command_kernel` is batchable, it is batched in a
    `cvk_command_batch`
- `clFlush` calls `cvk_command_queue::flush`
  - `m_executor` is initialized to `get_thread_pool()->get_executor()`
    - `clvk_global_state::init_executors` has created a
      `cvk_executor_thread_pool` during init
  - `m_executor->send_group` sends the cmd group to the executor
  - each `cvk_executor_thread` is a thread running
    `cvk_executor_thread::executor`
    - it pops the next `cvk_command_group` from `m_groups`
    - it calls `cvk_command_group::execute_cmds` to execute the cmds
- `clFinish` calls `cvk_command_queue::finish`
  - it flushes as usual
  - `cvk_command_queue::execute_cmds_required_by_no_lock` steals cmds from the
    executor and executes them directly

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
      - it replaces cl builtins by glsl natives
        - e.g., `Builtins::kAbs` by `glsl::ExtInst::ExtInstSAbs`
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
      - it lowers cl builtins by llvm instrs/intrinsics
        - e.g., `Builtins::kMad` by `Instruction::FMul` and
          `Instruction::FAdd`
    - `clspv::FixupBuiltinsPass`
    - `clspv::ThreeElementVectorLoweringPass`
    - `clspv::LongVectorLoweringPass`
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
