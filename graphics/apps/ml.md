Machine Learning
================

## Background

- <https://en.wikipedia.org/wiki/History_of_artificial_intelligence>
  - Birth of artificial intelligence (1941-56)
    - Alan Turing
    - artificial neural network (ANN)
    - symbolic reasoning
    - dartmouth workshop
  - Early successes (1956-1974)
    - ANN up to 4 layers
  - First AI winter (1974–1980)
    - limited compute power
  - Boom (1980–1987)
    - expert systems
    - ANN with backpropagation
  - Bust: second AI winter (1987–1993)
  - AI (1993–2011)
    - thanks to increased compute power and focus on specific domains
    - deep blue
    - probabilistic reasoning
  - Deep learning, big data (2011–2020)
  - Large language models, AI era (2020–present)
- <https://en.wikipedia.org/wiki/Timeline_of_machine_learning>
  - 1950s: machine learning pioneered
  - 1960s: probabilistic inference with bayesian methods 
  - 1970s: ai winter
  - 1980s: backpropagation, reinforcement learning
  - 1990s: Support-vector machines (SVMs), recurrent neural networks (RNNs)
  - 2000s: unsupervised machine learning
  - 2010s: deep learning
  - 2020s: large language models (LLMs) and generative AI
- <https://en.wikipedia.org/wiki/Deep_learning>
  - artificial neural network with multiple layers
  - deep neural networks
    - feedforward neural networks (FNNs)
    - recurrent neural networks (RNNs)
    - convolutional neural networks (CNNs)
    - transformers
  - training
    - supervised learning
    - unsupervised learning
    - reinforcement learning
- <https://en.wikipedia.org/wiki/Transformer_(deep_learning_architecture)>
  - primary components
    - tokenizer converts input into tokens
      - e.g., a sentence is broken down into an array of words or subwords
      - each word or subword is a token
    - an embedding layer converts tokens to vectors
      - token embedding: each token is converted to a vector individually
      - positional embedding: each token is converted to a vector based on its
        relative position in the token array
      - for each token, the two vectors are summed together to form the final
        vector
      - if there are M tokens and each token is converted to a N-dim vector,
        we have a MxN matrix
    - tranformer layers transform vectors repeatedly
      - the transformer layers can either be an encoder or a decoder
    - an optional un-embedding layer converts vectors to tokens
  - each encoder layer has two major components
    - a self-attention mechanism, which weights the input vectors
    - a feed-forward neural network, which process input vector individually
    - the resulting vectos are passed down to the next encoder layer
    - I guess this is the process of understanding the input, or "extracting
      the features"
      - the extracted features will be fed into the decoders
  - each decoder layer has three major components
    - a self-attention mechanism, just like an encoder
    - an attention mechanism, which takes inputs from encoders
    - a feed-forward neural network, just like an encoder
    - I guess this is the process of generating the output
      - the output tokens are generated one-by-one
      - the first output token is generated solely from the extracted features
      - the following output tokens are geneated based on the extracted
        features as well as prior output tokens
- <https://en.wikipedia.org/wiki/Large_language_model>
  - transformer-based
  - Google BERT, LaMDA, PaLM, and Gemini
  - OpenAI GPT-{1,2,3,3.5,4}
  - Meta Llama

## 3Blue1Brown: Deep Learning

- <https://www.youtube.com/watch?v=aircAruvnKk>
  - a neutral network to recognize the digit in a 28x28 image
  - first layer has `28x28=784` nodes, representing the intensity of each
    pixel
  - second layer has 16 nodes
    - `f(W * I + B)`
    - `W`, weights, is a 16x784 matrix
    - `I`, inputs, is a 784x1 column vector
    - `B`, biases, is a 16x1 column vector
    - `f` scales the results to `[0, 1]`
  - third layer has 16 nodes
    - `W` is a 16x16 matrix
    - `I` is a 16x1 column vector
    - `B` is a 16x1 column vector
  - last layer has 10 nodes, representing 0 to 9
    - `W` is a 10x16 matrix
    - `I` is a 16x1 column vector
    - `B` is a 10x1 column vector
  - in total, there are `(16*784+16) + (16*16+16) + (10*16+10) = 13002`
    parameters to be determined
- <https://www.youtube.com/watch?v=aircAruvnKk>
  - cost
    - randomly pick the 13002 parmeters first
    - given a sample image, the network can output a 10x1 column vector
    - since we know the correct digit, we know the correct 10x1 column vector
    - this allows us to calculate the cost, which is the distance between the
      two vectors
  - cost function
    - given a training set, we can run the entire training set through the
      network and calculate the average cost
    - this gives as a cost function, which maps 13002 parameters to a single
      value
    - the goal is to find the local minimum of the cost function
  - gradient descent to find local minimum of the cost function
    - when there is one parameter, `f(x) = y`
      - we can start with a randomly picked `x0`
      - we find the slope of the function at `x0`
      - if the slope is positive, we pick `x1` that is smaller than `x0`
      - if the slope is negative, we pick `x1` that is larger than `x0`
      - if the absolute value of the slope is large, `x1` is picked such that
        the distance to `x0` is large
      - we can repeat this process and find a local minimum
    - when there are 13002 parameters, the idea is the same

## Netron

- <https://netron.app/> is a model visualizer
- e.g.,
  - <https://storage.googleapis.com/chromiumos-test-assets-public/tast/cros/mlbenchmark/ml-test-assets-0.0.6_20230921.tar.gz>
  - `convolution_benchmark_288_512_1.tflite`
    - input: `float32[1,288,512,3]`
    - output: `float32[1,288,512,3]`

## TensorFlow Lite

- build with cmake
  - `git clone https://github.com/tensorflow/tensorflow.git`
  - `cmake -S tensorflow/lite -B out -G Ninja -DTFLITE_ENABLE_GPU=ON -DCMAKE_BUILD_TYPE=Release`
  - `ninja -C out tensorflow-lite benchmark_model`
  - `scp -C out/tools/benchmark/benchmark_model dut:`, or
  - `tflitedir=tflite-$(date +%Y%m%d) && tar -zcf $tflitedir.tar.gz --transform="s,^,$tflitedir/," -C out/tools/benchmark benchmark_model`
- delegates
  - `coreml`, Core ML, Apple
  - `gpu`, CL/GL/Metal
  - `hexagon`, Hexagon, Qualcomm
  - `nnapi`, NNAPI, Android
  - `xnnpack`, XNNPACK, x86/ARM/WebAssembly
    - <https://github.com/google/XNNPACK>
  - external delegates
    - TPU, NPU, WebGPU ,etc.
- `benchmark_model`
  - it seems after initial setup and inference, each inference does
    - `clEnqueueWriteBuffer`
    - `clSetKernelArg` and `clEnqueueNDRangeKernel` for
      - `bhwc_to_tensor` once
      - `main_function` many times
      - `clFlush`
      - `tensor_to_bhwc` once
    - `clEnqueueReadBuffer`
    - `clFinish`
  - gpu time breakdown
    - `clEnqueueWriteBuffer`: 2.5%
    - `clEnqueueNDRangeKernel(bhwc_to_tensor)`: 1.5%
    - `clEnqueueNDRangeKernel(main_function)`: 84%
    - `clEnqueueNDRangeKernel(tensor_to_bhwc)`: 1%
    - `clEnqueueReadBuffer`: 11%
  - clvk
    - `clEnqueueWriteBuffer` and `clEnqueueReadBuffer` are handled in
      `cvk_mem::copy_from` and `cvk_mem::copy_to`
      - `vkMapMemory`
      - memcpy
      - `vkUnmapMemory`
    - `clEnqueueNDRangeKernel` are batched and are handled in
      `cvk_command_kernel::build_batchable_inner`
      - `cvk_kernel_argument_values::setup_descriptor_sets`
        - `vkAllocateDescriptorSets`
        - `vkUpdateDescriptorSets`
      - `vkCmdBindDescriptorSets`
      - `cvk_command_kernel::update_global_push_constants`
        - `vkCmdPushConstants`
      - `cvk_command_kernel::dispatch_uniform_region_within_vklimits`
        - `vkCmdBindPipeline`
        - `vkCmdPushConstants`
        - `vkCmdDispatch`
      - `vkCmdPipelineBarrier`
    - `clFlush` is handled in `cvk_command_queue::flush_no_lock`
      - `cvk_command_batchable::do_action`
        - `vkQueueSubmit`
        - `vkQueueWaitIdle`
        - yeah, clvk follows all submits by waits...
  - basic flow
    - `BenchmarkModel::Run` is the entrypoint
    - `BenchmarkTfLiteModel::ValidateParams` validates params
    - `BenchmarkTfLiteModel::LogParams` prints params
    - `BenchmarkTfLiteModel::Init` initializes
      - `BenchmarkTfLiteModel::LoadModel` loads the model
        - `tflite::FlatBufferModel::BuildFromBuffer` creates a
          `tflite::FlatBufferModel`
      - `BenchmarkTfLiteModel::InitInterpreter`
        - `tflite::InterpreterBuilder` creates a `tflite::Interpreter`
      - `ProvidedDelegateList::CreateAllRankedDelegates` creates delegates
        - `GpuDelegateProvider::CreateTfLiteDelegate` calls
          `TfLiteGpuDelegateV2Create` indirectly to create a `TfLiteDelegate`
      - `tflite::InterpreterBuilder::ModifyGraphWithDelegate` modifies the graph
        - `tflite::Subgraph::ModifyGraphWithDelegateImpl` calls
          `TfLiteDelegatePrepareInternal` to prepare the delegate
        - `tflite::gpu::DelegatePrepare` is the callback
      - `tflite::InterpreterBuilder::AllocateTensors` allocates tensors
    - `BenchmarkTfLiteModel::PrepareInputData` prepares input data
      - `CreateRandomTensorData` generates random input data for each input
        tensor
    - `BenchmarkModel::Run` is called twice
      - first time is warmup
      - `BenchmarkTfLiteModel::ResetInputsAndOutputs` resets inputs/outputs
        for each iteration
        - it copies the generated random input data to `t->data.raw`
      - `BenchmarkTfLiteModel::RunImpl` calls `tflite::Interpreter::Invoke`
        for each iteration
- gpu cl delegate
  - `TfLiteGpuDelegateV2Create` creates a `tflite::gpu::Delegate`
  - on `tflite::InterpreterBuilder::ModifyGraphWithDelegate`,
    `TfLiteDelegatePrepareInternal` is indirectly called
    - `tflite::gpu::DelegatePrepare` is the callback
    - `tflite::Subgraph::ReplaceNodeSubsetsWithDelegateKernels` calls
      `tflite::Subgraph::AddNodeWithParameters` with the registration created
      by `tflite::gpu::CreateRegistration`
      - it calls `init` callback of the registration
        - this creates and prepares a `tflite::gpu::DelegateKernel`
        - `DelegateKernelCore::Prepare` calls `DelegateKernelCore::Setup`
          which calls `DelegateKernelCore::InitializeOpenClApi`
        - `tflite::gpu::cl::NewInferenceEnvironment` creates a
          `tflite::gpu::cl::InferenceEnvironmentImpl`
          - this is where cl is initialized
        - `tflite::gpu::cl::InferenceEnvironmentImpl::NewInferenceBuilder`
          creates and inits a `tflite::gpu::cl::InferenceBuilderImpl`
          - `tflite::gpu::cl::InferenceContext::InitFromGraph` calls
            `tflite::gpu::cl::InferenceContext::AllocateMemory` to create cl
            buffers and images
          - it also calls `tflite::gpu::cl::InferenceContext::Tune` to find
            the best work group sizes for the operations
          - `tflite::gpu::cl::InferenceContext::Compile` creates more cl
            buffers and creates cl programs
            - `tflite::gpu::ConvertOperations` has called
              `GPUOperationFromNode` to convert nodes to operations and
              `tflite::gpu::ConvGeneric::GenerateCode` has generated the
              generic convert code
            - `tflite::gpu::cl::ClOperation::Compile` adds defines for the
              kernel source and builds it
        - `tflite::gpu::cl::InferenceBuilderImpl::Build` calls
          `tflite::gpu::cl::InferenceRunnerImpl::LinkTensors`
          - `tflite::gpu::cl::DefaultTensorTie::New` calls
            `tflite::gpu::cl::OpenClTensorConverterBuilder::MakeConverter` to
            create
            - `tflite::gpu::cl::BHWCBufferToTensorConverter::Init` generates
              the kernel source and build it
              - `CreateBhwcBufferToTensorOp` returns the source template
            - `tflite::gpu::cl::TensorToBHWCBufferConverter::Init` generates
              the kernel source and build it
              - `CreateTensorToBhwcBufferOp` returns the source template
          - it also calls
            `tflite::gpu::cl::DefaultTensorTie::MaybeAllocateExternalObject`
            to allocate cl buffer
  - on `tflite::impl::Interpreter::Invoke`, the `invoke` callback of the
    registration created by `tflite::gpu::CreateRegistration` is called
    - it calls `tflite::gpu::DelegateKernel::Invoke`
    - `tflite::gpu::cl::InferenceRunnerImpl::Run` does
      - `CopyFromExternalObject`
        - `tflite::gpu::cl::CpuCopier::Convert` calls `clEnqueueWriteBuffer`
        - `tflite::gpu::cl::BHWCBufferToTensorConverter::Convert` calls
          `clEnqueueNDRangeKernel`
      - `RunWithoutExternalBufferCopy`
        - `tflite::gpu::cl::InferenceContext::AddToQueue` calls
          `tflite::gpu::cl::CLCommandQueue::Dispatch` which calls
          `clEnqueueNDRangeKernel`
        - it also calls `clFlush` directly at the end
      - `CopyToExternalObject`
        - `tflite::gpu::cl::TensorToBHWCBufferConverter::Convert` calls
          `clEnqueueNDRangeKernel`
        - `tflite::gpu::cl::CpuCopier::Convert` calls `clEnqueueReadBuffer`
      - `WaitForCompletion` calls `clFinish` directly
- `tflite::gpu::ConvGeneric::GuessBestParams`
  - common inputs are
    - `dst_shape` is BHWC (batch, height, width, channel)
      - batch and channel are usually 1
      - width and height depend on model
    - some common values are
      - `512x288` with src/dst depth `1x2` or `6x2` or `2x1`
      - `256x144` with src/dst depth `2x4` or `12x4`
      - `128x72` with src/dst depth `4x8` or `24x8`
      - `64x36` with src/dst depth `8x16` or `48x16`
      - `32x18` with src/dst depth `16x32`
      - I guess when the input size is small, it increases depth to process
        more slices at a time
    - `x_kernel_is_1` and `y_kernel_is_1` are usually false, unless the shape
      width/height are small
    - `different_weights_for_height` is false most of the time
  - assume
    - `(src_depth, dst_depth)` is `(6, 2)`
    - `x_kernel_is_1` and `y_kernel_is_1` are false
    - `different_weights_for_height` is false
    - `dst_shape` width and height are `512x288`
  - AMD uses
    - `conv_params.weights_data_type` is `DataType::FLOAT16`
    - `conv_params.block_size` is `int4(4, 2, 1, 2)`
      - this is the number of outputs to compute per work item
      - when `dst_depth` is a multiple of 4, it switches to
        `int4(2, 2, 1, 4)`, etc.
    - `conv_params.fixed_work_group_size` is false
    - `conv_params.work_group_size` is undefined and unused
    - `conv_params.work_group_launch_order` is undefined and unused
    - `conv_params.linear_spatial` is false
    - `conv_params.linear_all` is false
    - `conv_params.different_weights_for_height` is false
    - `conv_params.groups_support` is false (default)
    - `conv_params.src_depth_loop_size` is 1
      - this is the number of slices to process in the inner most loop
      - `__attribute__((opencl_unroll_hint(src_depth_loop_size)))`
        conceptually
    - `conv_params.need_src_loop` is true (default)
    - `conv_params.need_dst_loop` is true (default)
    - `conv_params.weights_upload_type` is `WeightsUploadType::CONSTANT_MEM`
      - this is the address space of the weights
    - `conv_params.x_kernel_is_1` is false
    - `conv_params.y_kernel_is_1` is false
    - `conv_params.z_kernel_is_1` is false
      - these are the kernel size (as in a box filter)
    - `conv_params.weights_layout` is `WeightsLayout::kOSpatialIOGroupI4O4`
    - `conv_params.simd_size` is 1 (default)
    - `work_group_size_` is `int3(8, 4, 1)`
      - this is the local work group size
    - `work_group_launch_order_` is `int3(0, 1, 2)`
  - Intel uses
    - `conv_params.weights_data_type` is `DataType::FLOAT16`
    - `conv_params.block_size` is `int4(1, 1, 1, 2)`
      - when `dst_depth` is a multiple of 4, it switches to
        `int4(1, 1, 1, 4)`, etc.
    - `conv_params.fixed_work_group_size` is true
    - `conv_params.work_group_size` is undefined and unused
    - `conv_params.work_group_launch_order` is undefined and unused
    - `conv_params.linear_spatial` is true
    - `conv_params.linear_all` is false
    - `conv_params.different_weights_for_height` is false
    - `conv_params.groups_support` is false (default)
    - `conv_params.src_depth_loop_size` is 2
      - when `src_depth` is a multiple of 4, it switches to 4
    - `conv_params.need_src_loop` is true (default)
    - `conv_params.need_dst_loop` is true (default)
    - `conv_params.weights_upload_type` is `WeightsUploadType::PRIVATE_MEM_SIMD_BROADCAST`
    - `conv_params.x_kernel_is_1` is false
    - `conv_params.y_kernel_is_1` is false
    - `conv_params.z_kernel_is_1` is false
    - `conv_params.weights_layout` is `WeightsLayout::kOSpatialIOGroupI4O4`
    - `conv_params.simd_size` is 16
      - it will be 32 if we use subgroup on amd
    - `work_group_size_` is `int3(16, 1, 1)`
      - it will be 32 if we use subgroup on amd
    - `work_group_launch_order_` is `int3(0, 1, 2)`
- `tflite::gpu::ConvGeneric::GenerateConv`
  - the goal is to compute `Output = Weight * Input + Bias` with box filtering
  - there are `src_tensor.Width * src_tensor.Height * src_tensor.Slices` input values
    - the type is `half4` (4 channels)
  - there are `dst_tensor.Width * dst_tensor.Height * dst_tensor.Slices` output values
    - the type is also `half4`
  - there are `dst_tensor.Slices` biases
    - the type is `half4`
    - all output values for the same dst slice will be biased by the same bias
  - there are `kernel_size_x * kernel_size_y * src_tensor.Slices * dst_tensor.Slices`
    weight matrices
    - the type is 4x4 matrix
  - each output value is the average of intermediate output values, plus bias
    - there are `kernel_size_x * kernel_size_y * src_tensor.Slices * dst_tensor.Slices`
      intermediate output values
    - each intermediate output value is calculated as `weight * input`
      - `input` varies for different `(x, y, src_slice)`
      - `weight` varies for different `(kernel_x, kernel_y, src_slice, dst_slice)`
  - each work item handles `conv_params.block_size` outputs
    - e.g., when the block size is `int4(4, 2, 1, 2)`, each work items outputs
      `4*2*1*2=16` values
      - they are `r_wX_hY_sW` (no Z because it is 1)

## MediaPipe Solution Example

- install
  - `pip install -q mediapipe`
  - `wget -O deeplabv3.tflite -q https://storage.googleapis.com/mediapipe-models/image_segmenter/deeplab_v3/float32/1/deeplab_v3.tflite`
- load image
  - `import mediapipe as mp`
  - `image = mp.Image.create_from_file('test.jpg')`
- create segmenter
  - `from mediapipe.tasks import python`
  - `from mediapipe.tasks.python import vision`
  - `base_options = python.BaseOptions(model_asset_path='deeplabv3.tflite')`
  - `options = vision.ImageSegmenterOptions(base_options=base_options, output_category_mask=True)`
  - `segmenter = vision.ImageSegmenter.create_from_options(options)`
- run
  - `result = segmenter.segment(image)`

## MediaPipe Framework Example

- install
  - dependencies
    - download `https://github.com/bazelbuild/bazelisk` and add it to `PATH`
    - `apt install libopencv-video-dev libopencv-contrib-dev` for opencv (and
      ffmpeg)
    - `apt install libgles-dev` for gles2
  - `git clone --depth 1 https://github.com/google/mediapipe.git`
  - edit `mediapipe/third_party/opencv_linux.BUILD`
    - for opencv 4.x headers and includes
  - hello world (on cpu)
    - `bazel run --define MEDIAPIPE_DISABLE_GPU=1 mediapipe/examples/desktop/hello_world:hello_world`
  - hello world (on gpu)
    - `bazel run --copt -DMESA_EGL_NO_X11_HEADERS --copt -DEGL_NO_X11 mediapipe/examples/desktop/hello_world:hello_world`
- <https://github.com/google/mediapipe/blob/master/mediapipe/examples/desktop/hello_world/hello_world.cc>
  - create a `CalculatorGraphConfig`
    - `ParseTextProtoOrDie` parses a pbtxt into a `CalculatorGraphConfig`
    - the proto is defined at
      <https://github.com/google/mediapipe/blob/master/mediapipe/framework/calculator.proto>
    - <https://viz.mediapipe.dev/> is a visualizer
    - this describes a graph in protobuf
      - each node is a calculator
      - each edge is a stream
  - init a `CalculatorGraph` from the `CalculatorGraphConfig`
  - add a `OutputStreamPoller` for the output stream
  - run the graph async
  - add packets to the input stream
    - the packets consist of a string and a timestamp
      `MakePacket<std::string>("Hello World!").At(Timestamp(i))`
  - close the input stream
  - use `OutputStreamPoller` to get the packets from the output stream
    - because the calculators are `PassThroughCalculator`, the output packets
      have the same contents as the input packets
  - wait for the graph to be done
- <https://developers.google.com/mediapipe/framework/framework_concepts/overview>
- calculators
  - `BypassCalculator` is a passthrough calculator with the ability to remap
    input/output streams
  - `GpuBufferToImageFrameCalculator` and `ImageFrameToGpuBufferCalculator`
    convert between `GpuBuffer` for gpu-based calculators and `ImageFrame` for
    cpu-based calculators
  - tflite example
    - <https://github.com/google/mediapipe/blob/master/mediapipe/graphs/hair_segmentation/hair_segmentation_mobile_gpu.pbtxt>
    - `TfLiteConverterCalculator` converts `ImageFrame`/`Matrix` to
      `TfLiteTensor` and converts `GpuBuffer` to `tflite::gpu::GlBuffer`
    - `TfLiteInferenceCalculator` runs inference on the tflite tensors and
      tflite model
    - `TfLiteTensorsToSegmentationCalculator` converts `TfLiteTensor` into a
      image mask
