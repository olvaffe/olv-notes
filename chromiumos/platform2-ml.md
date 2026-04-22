CrOS ML
=======

## ML Service

- <https://chromium.googlesource.com/chromiumos/platform2/+/main/ml>
  - it provides `ml_service` daemon
  - it uses tflite
  - it is built by `chromeos-base/ml`

## ML Core

- unrelated to ML service
- <https://chromium.googlesource.com/chromiumos/platform2/+/main/ml_core>
  - it provides `libcros_ml_core` library
  - it dlopens proprietary `libcros_ml_core_internal` library
  - it is built by `dev-libs/ml-core`
- it only provides `EffectsPipeline` which is used by the camera service

## TensorFlow Lite

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

