MediaPipe
=========

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
