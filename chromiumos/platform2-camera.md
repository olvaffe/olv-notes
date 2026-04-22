Platform2 Camera
================

## Overview

- <https://chromium.googlesource.com/chromiumos/platform2/+/refs/heads/main/camera/README.md>

## ebuilds

- `virtual/cros-camera-hal` depends on the board-specific hal ebuild
  - such as `media-libs/cros-camera-hal-usb`
- `virtual/cros-camera-hal-configs` depends on the board-specific hal config
  ebuild
  - `media-libs/cros-camera-hal-usb` parses
    `/etc/camera/camera_characteristics.conf`,
    `/run/camera/camera_config.json`, etc.
- `cros-camera-hal-usb`
  - source `PLATFORM_SUBDIR="camera/hal/usb"`
  - binaries are `/usr/lib64/camera_hal/usb.so`
- `cros-camera-libfs`
  - prebuilt binaries include
    `/build/share/cros_camera/libdocumentscanner.so`,
    `/build/share/cros_camera/libfacessd_cros.so`, etc.
- `cros-camera-frame-annotator`
  - source `PLATFORM_SUBDIR="camera/features/frame_annotator/libs"`
  - binaries are `/usr/lib64/libcros_camera_frame_annotator.so`
- `cros-camera-android-deps`
  - source `PLATFORM_SUBDIR="camera/android"`
  - binaries are `/usr/lib64/libcros_camera_android_deps.so` and headers for
    android camera3 hal
- `cros-camera`
  - source `PLATFORM_SUBDIR="camera/hal_adapter"`
  - binaries are `/usr/bin/cros_camera_service`
- `cros-camera-libs`
  - source `PLATFORM_SUBDIR="camera/common"`
  - binaries are `/usr/bin/cros_camera_algo`, `/usr/lib64/libcros_camera.so`,
    etc.
- `ml-core`
  - source `PLATFORM_SUBDIR="ml_core"`
  - binaries are `/usr/lib64/libcros_ml_core.so`
  - prebuilt binaries are `/build/share/ml_core/libcros_ml_core_internal.so`
- `cros-camera-gpu-test`
  - source `PLATFORM_SUBDIR="camera/gpu/tests"`
  - binaries are `/usr/bin/image_processor_test`
- `cros-camera-gl-loader`
  - source `PLATFORM_SUBDIR="camera/gpu/gl_loader"`
  - binaries are
    - `/usr/lib64/libcamera_egl_loader.so`
    - `/usr/lib64/libcamera_gles_loader.so`
    - symlinks under `/usr/lib64/camera_gl_loader`
  - when `USE=camera_angle_backend`
    - other daemons add `/usr/lib64/camera_gl_loader` to their rpath to use
      the loader
    - the loader dlopens `/usr/lib64/angle/libEGL.so`, which is provided by
      `chromeos-chrome`
      - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/bf1faee2c81e1d8c9f6a83e8760b728108960289>

## mojo

- <https://chromium.googlesource.com/chromiumos/platform2/+/refs/heads/main/camera/mojo/camera3.mojom>
  - `Camera3DeviceOps` documents the steps
    - `Initialize` registers `Camera3CallbackOps` with the camera hal
    - `ConfigureStreams` negotiates the usable streams
    - `ConstructDefaultRequestSettings` gets the capture settings
    - capture loop
      - `ProcessCaptureRequest` requests the camera hal to do a capture
      - the camera hal calls `Notify` and `ProcessCaptureResult`
    - `Dump` dumps debug info
    - `Flush` tells the camera hal to finish/discard ongoing capture requests
      and to return to the idle state
    - `Close` closes the camera device

## USB HAL

- initialization
  - the hal exposes these public symbols
    - `CROS_CAMERA_HAL_INFO_SYM`
    - `HAL_MODULE_INFO_SYM`
    - all calls are dispatched to `CameraHal::GetInstance()`
  - `CameraHal::Init`
    - `UdevWatcher::EnumerateExistingDevices` calls `CameraHal::OnDeviceAdded`
      on all devices
      - `V4L2CameraDevice::IsCameraDevice` checks if the device is a camera
      - the device is added to `device_infos_`
  - `CameraHal::OpenDevice` creates a `CameraClient` and calls
    `CameraClient::OpenDevice`
    - this creates a `V4L2CameraDevice` and calls various methods
- `V4L2CameraDevice`
  - `V4L2CameraDevice::GetDeviceSupportedFormats`
    - this opens the devnode and calls `VIDIOC_ENUM_FMT`,
      `VIDIOC_ENUM_FRAMESIZES`, and `VIDIOC_ENUM_FRAMEINTERVALS` to enumerate
      all formats for buf type `V4L2_BUF_TYPE_VIDEO_CAPTURE`
  - `V4L2CameraDevice::Connect`
    - this opens the devnode
    - it does `VIDIOC_G_FMT` and `VIDIOC_S_FMT` to "own" the device
  - `V4L2CameraDevice::StreamOn`
    - it calls `VIDIOC_S_FMT` to set the format
    - it calls `VIDIOC_REQBUFS` to allocate buffers
      - the type is `V4L2_BUF_TYPE_VIDEO_CAPTURE`
      - the memory is `V4L2_MEMORY_MMAP`
      - the count is `kNumVideoBuffers` (4)
    - it calls `VIDIOC_EXPBUF` to export buffers as dmabufs
    - it calls `VIDIOC_QBUF` to queue buffers
    - it calls `VIDIOC_STREAMON` to start capturing
  - `V4L2CameraDevice::StreamOff`
    - it calls `VIDIOC_STREAMOFF` to stop capturing
    - it calls `VIDIOC_REQBUFS` to free buffers
  - `V4L2CameraDevice::IsBufferFilled`
    - it calls `VIDIOC_QUERYBUF` to check if a buffer is ready for readback
  - `V4L2CameraDevice::GetNextFrameBuffer`
    - it calls `VIDIOC_DQBUF` to dequeue a buffer that is ready for readback
  - `V4L2CameraDevice::ReuseFrameBuffer`
    - it calls `VIDIOC_QBUF` to queue a buffer again after readback
- `CameraClient`
  - `CameraClient::ConfigureStreams` starts capturing
    - `CameraClient::RequestHandler::StreamOnImpl` calls
      `V4L2CameraDevice::StreamOn` and remebers the dmabufs in
      `input_buffers_`
  - `CameraClient::ProcessCaptureRequest` writes captured frames to
    `output_buffers`
    - `output_buffers` are gbm bos with associated fences
    - `CameraClient::RequestHandler::DequeueV4L2Buffer` calls
      `V4L2CameraDevice::GetNextFrameBuffer`
    - `CameraClient::RequestHandler::WriteStreamBuffers` copies from the
      dmabuf input to gbm bo output
      - the dmabuf input is wrapped in `V4L2FrameBuffer`
      - the gbm bo output output is wrapped in `GrallocFrameBuffer`
      - `CachedFrame::Convert` does the coversion/copying
    - `CameraClient::RequestHandler::EnqueueV4L2Buffer` calls
      `V4L2CameraDevice::ReuseFrameBuffer`
- `CachedFrame::Convert`
  - the dmabuf input is decoded or converted to NV12
    - `CachedFrame::DecodeToNV12` tries JDA (chrome jpeg decode accel) first
      and falls back to `ImageProcessor::ConvertFormat`
    - `ImageProcessor::ConvertFormat` uses libyuv
  - the gbm bo output is encoded or converted from NV12
    - `CachedFrame::ConvertFromNV12`

## Effects

- `CameraHalAdapter::OpenDevice` creates a `CameraDeviceAdapter` with a
  `StreamManipulatorManager`, and forwards many calls to the stream
  manipulator manager
  - `CameraDeviceAdapter::Initialize` calls
    `StreamManipulatorManager::Initialize`
  - `CameraDeviceAdapter::ConfigureStreams` calls
    `StreamManipulatorManager::ConfigureStreams` and
    `StreamManipulatorManager::OnConfiguredStreams`
  - `CameraDeviceAdapter::ConstructDefaultRequestSettings` calls
    `StreamManipulatorManager::ConstructDefaultRequestSettings`
  - `CameraDeviceAdapter::ProcessCaptureRequest` calls
    `StreamManipulatorManager::ProcessCaptureRequest`
  - `CameraDeviceAdapter::Flush` calls `StreamManipulatorManager::Flush`
  - `CameraDeviceAdapter::Close` calls
    `StreamManipulatorManager::StopProcessing`
  - `CameraDeviceAdapter::ProcessCaptureResult` calls
    `StreamManipulatorManager::ProcessCaptureResult`
  - `CameraDeviceAdapter::Notify` calls `StreamManipulatorManager::Notify`
- `EffectsGlThread` thread
  - `EffectsStreamManipulatorImpl::SetupGlThread` sets up an EGL context and
    `GpuImageProcessor`
  - `EffectsStreamManipulatorImpl::ProcessCaptureResult` calls
    `EffectsStreamManipulatorImpl::RenderEffect`
  - `EffectsStreamManipulatorImpl::RenderEffect`
    - calls `SharedImage::CreateFromBuffer` to create `yuv_image` from the
      capture result
    - calls `CameraBufferPool::RequestBuffer` to allocate a `rgba_buffer`
    - calls `SharedImage::CreateFromBuffer` to create `rgba_image` from
      `rgba_buffer`
    - calls `GpuImageProcessor::NV12ToRGBA` followed by `glFinish` to convert
      yuv to rgb
    - calls `EffectsPipeline::ProcessFrame` to process the rgb image
  - `EffectsStreamManipulatorImpl::OnFrameProcessed` calls
    `EffectsStreamManipulatorImpl::PostProcess` after the pipeline processes
    the rgb image
  - `EffectsStreamManipulatorImpl::PostProcess` calls
    `GpuImageProcessor::RGBAToNV12` followed by `glFinish` to convert rgb to
    yuv
    - the process context is destroyed at the end of the function
    - this destroys `yuv_image` and `rgba_image`
- `SharedImage::CreateFromBuffer`
  - `EglImage::FromBuffer` calls `eglCreateImage` to create `EGLImage`s from
    external buffers
  - `Texture2D` ctor with `EGLImage` calls `glGenTextures` and
    `glEGLImageTargetTexture2DOES` to create `GL_TEXTURE_2D` texture objects

## Effects Chrome Trace

- I have a trace for chrome showing camera preview with effects
- `cros_camera_service`: `Capture request` wakes up and calls
  `CameraDeviceAdapter::ProcessCaptureResult`
- `cros_camera_service`: `DefaultCaptureR` wakes up and calls
  `StreamManipulatorManager::ProcessCaptureResultOnStreamManipulator`
- `cros_camera_service`: `EffectsGlThread` wakes up and calls
  `EffectsStreamManipulatorImpl::ProcessCaptureResult` which calls
  `EffectsStreamManipulatorImpl::RenderEffect`
  - it calls `GpuImageProcessor::NV12ToRGBA` followed by `glFinish` to convert yuv to
    rgb
  - it calls `EffectsPipeline::ProcessFrame` which is another beast
    - it seems there are multiple `drishti_gl_runn` threads and most
      PROCESSes are on diffrent threads
      - `drishti` should be the code name for mediapipe
      - `PROCESS::ImageHasMinWidthAndHeightCalculator`
      - `PROCESS::optionalimagedownscalecontainer`
      - `PROCESS::syrtisvulkansegmentationsubgraph`
      - `PROCESS::autolightpositionsubgraph`
      - `PROCESS::relightcontainer`
      - `PROCESS::bgreplacecontainer`
      - `PROCESS::bgblurcontainer`
      - `PROCESS::waitonrendercontainer`
      - `EffectsStreamManipulatorImpl::OnFrameProcessed`
    - it seems the effects pipeline has a queue.  `ProcessFrame` can submit
      framen N while the following `OnFrameProcessed` returns frame N-1 or
      even N-2.  For example,
      - 0.0ms: `ProcessFrame(N)`
      - 0.2ms-0.5ms: `PROCESS::ImageHasMinWidthAndHeightCalculator(N)` and
                     `PROCESS::optionalimagedownscalecontainer(N)`
      - 9.3ms: `PROCESS::waitonrendercontainer(N-1)` ends and
        `PROCESS::syrtisvulkansegmentationsubgraph(N)` begins
      - 9.5ms: `OnFrameProcessed(N-1)`
      - 23.0ms: `PROCESS::syrtisvulkansegmentationsubgraph(N)` ends
        - the gpu time is 9ms
      - 34.0ms-51.3ms: `PROCESS::relightcontainer(N)`
        - 1x `clEnqueueWriteBuffer`
        - Nx `clEnqueueNDRangeKernel` and 1x `clFlush`
          - 2x `vkQueueSubmit` on clvk
        - 1x `clEnqueueReadBuffer`
          - 1x `vkQueueSubmit` on clvk
        - the gpu times of the 3 queue submits are 4ms, 2ms, 5ms
      - 51.3ms-51.7ms: `PROCESS::bgreplacecontainer(N)` and `PROCESS::bgblurcontainer(N)`
      - 72.2ms: `PROCESS::waitonrendercontainer(N)` ends
  - on effects pipeline completion,
    `EffectsStreamManipulatorImpl::PostProcess` is called
    - this calls `GpuImageProcessor::RGBAToNV12` followed by `glFinish` to
      convert rgb back to yuv
  - on skyrim, it easily takes >30ms
