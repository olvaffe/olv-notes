CrOS TensorFlow Lite
====================

## Build

- `emerge-$BOARD sci-libs/tensorflow`
  - `platform/tflite` is the source repo
    - it contains only cros-specific code and patches
  - the majority of the source code are tarballs listed in
    `bazel_external_uris`; some interesting ones are from
    - <https://github.com/tensorflow/tensorflow>
    - <https://github.com/tensorflow/runtime>
    - <https://github.com/google/XNNPACK>
    - <https://github.com/google/ruy>
    - <https://github.com/jax-ml/ml_dtypes>
    - <https://gitlab.com/libeigen/eigen>
- build with cmake
  - `git clone https://github.com/tensorflow/tensorflow.git`
  - `cmake -S tensorflow/lite -B out -G Ninja -DTFLITE_ENABLE_GPU=ON`
  - `ninja -C out tensorflow-lite benchmark_model`
- installed files
  - `/usr/include/tensorflow`
    - `fp16`
    - `nnapi`
    - `ruy`
    - `tensorflow/core`
    - `tensorflow/lite`
  - `/usr/lib64`
    - `libnativewindow.so`
    - `libtensorflowlite.so`
    - `pkgconfig/tensorflowlite.pc`
  - `/usr/local/bin`
    - `android_hardware_buffer_test`
    - `async_delegate_test`
    - `benchmark_model`
    - `inference_diff_eval`
    - `object_detection_eval`
    - `stable_delegate_test_suite`
  - `/usr/local/lib64`
    - `libtensorflowlite_cros_sample_delegate.so`
- tast tests
  - `tast list <dut> | grep mlbenchmark`
    - `mlbenchmark.TFLite.*`
  - `tast list -buildbundle crosint <dut> | grep mlbenchmark`
    - `mlbenchmark.SODA.*`
    - `mlbenchmark.TFLiteInternal.*`
    - `mlbenchmark.ULM.*`
    - `mlbenchmark.ULMEstimator.*`

## `benchmark_model`

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

## Tast `mlbenchmark.TFLite.*`

- <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/bundles/cros/mlbenchmark/>
- test assets, `ml-test-assets.tar.gz`
  - it contains various `.tflite` files
  - it is unpacked to `/usr/local/mlbenchmark/data`
- the test runs `benchmark_model --min_secs=60 --report_peak_memory_footprint=true --graph=<path-to-tflite-file>`
  - this uses cpu
  - when the backend is `mlbenchmark.KGpuOpenCl`, `--use_gpu=true --gpu_backend=cl` are specified to use cl
- selected test outputs
  - if cpu
    - `INFO: Created TensorFlow Lite XNNPACK delegate for CPU.`
    - `VERBOSE: Replacing 58 out of 58 node(s) with delegate (TfLiteXNNPackDelegate) node, yielding 1 partitions for the whole graph.`
    - `INFO: Initialized session in 3.337ms.`
    - `INFO: count=9 first=81913 curr=55502 min=55502 max=81913 avg=58984.7 std=8115`
    - `INFO: count=1061 first=56576 curr=56299 min=54669 max=60236 avg=56265.8 std=982`
    - `INFO: Inference timings in us: Init: 3337, First inference: 81913, Warmup (avg): 58984.7, Inference (avg): 56265.8`
    - `INFO: Overall peak memory footprint (MB) via periodic monitoring: 93.2383`
  - if gpu
    - `INFO: Created TensorFlow Lite delegate for GPU.`
    - `VERBOSE: Replacing 58 out of 58 node(s) with delegate (TfLiteGpuDelegateV2) node, yielding 1 partitions for the whole graph.`
    - `INFO: Initialized OpenCL-based API.`
    - `INFO: Created 1 GPU delegate kernels.`
    - `INFO: Explicitly applied GPU delegate, and the model graph will be completely executed by the delegate.`
    - `INFO: Initialized session in 842.225ms.`
    - `INFO: count=13 first=115098 curr=36666 min=32079 max=115098 avg=40199.8 std=21694`
    - `INFO: count=1694 first=35688 curr=33419 min=30394 max=42335 avg=35134.9 std=1733`
    - `INFO: Inference timings in us: Init: 842225, First inference: 115098, Warmup (avg): 40199.8, Inference (avg): 35134.9`
    - `INFO: Overall peak memory footprint (MB) via periodic monitoring: 74.6641`
- tast parses the outputs for
  - `init_latency` is parsed from `Init:`
  - `first_inference_latency` is parsed from `First inference:`
  - `avg_latency` is parsed from `avg=`
  - `std_dev` is parsed from `std=`
  - `peak_memory` is parsed from the peak memory line