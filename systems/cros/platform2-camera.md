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
