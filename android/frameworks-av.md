# Android AV

## Virtual Camera

- `android.virtualdevice.cts.camera.VirtualCameraCameraXTest`
  - `setUp`
    - `androidx` is the modern toolkit for apps
    - `android.compaion` provides apis for companion (external) devices
    - `VirtualDeviceManager.VirtualDevice` is a virtual companion device
      - camera, display, audio, keyboard, mouse, etc.
    - `virtualDeviceRule` helper
      - `createManagedVirtualDevice` creates a virtual device
      - `createManagedVirtualDisplay` creates a virtual display
      - `startActivityOnDisplaySync` starts activity in the virtual display
  - `createVirtualCamera`
    - it builds a `VirtualCameraConfig`
      - input stream is 460x260, `ImageFormat.YUV_420_888`
      - `VirtualCameraCallback` is called to produce a new input frame
    - `VirtualDevice.createVirtualCamera` returns a new
      `android.companion.virtual.camera.VirtualCamera`
      - `mVirtualDevice.registerVirtualCamera` calls
        `VirtualCameraController.registerCamera` to start `virtual_camera`
        service on demand and register a camera
  - `initCameraXProvider` configures the `ProcessCameraProvider`
    - it manages opening and closing of the virtual camera, as a camera client
  - `imageCapture.takePicture` takes a picutre
- major components
  - `VirtualCameraService` provides the virtual camera system service
    - owner apps use the service to register virtual cameras
  - `VirtualCameraProvider` is a camera HAL `BnCameraProvider`
    - a virtual camera is emulated at the HAL level
    - `client app -> cameraserver -> VirtualCameraProvider`
    - a `VirtualCameraProvider` is registered on process start
  - `VirtualCameraDevice` is a camera HAL `BnCameraDevice`
    - `VirtualCameraService::registerCamera` calls
      `VirtualCameraProvider::createCamera` to create a device
  - `VirtualCameraSession` is a camera HAL `BnCameraDeviceSession`
    - `VirtualCameraDevice::open` creates a session
- `VirtualCameraSession::configureStreams`
  - this is called from
    - client app `cameraDevice.createCaptureSession`
    - framework `mRemoteDevice.endConfigure`
    - cameraservice `mDevice->configureStreams`
    - cameraservice `mInterface->configureStreams`
  - `in_requestedConfiguration` is from the client app
    - client app typically requests two streams
      - a preview stream of `IMPLEMENTATION_DEFINED` and `GPU_TEXTURE`
      - a capture stream of format `BLOB` and `CPU_READ_OFTEN`
  - `getHalStream` creates a `HalStream` for each requested streams
    - it overrides `IMPLEMENTATION_DEFINED` to `YCRCB_420_SP`
    - it adds its own usage, such as
      - `CAMERA_OUTPUT`
      - `CPU_WRITE_OFTEN`
      - `GPU_RENDER_TARGET`
  - `pickInputConfigurationForStreams` picks the input config
    - owner app typicallys provides `YUV_420_888`
  - it creates a `VirtualCameraRenderThread`
- `VirtualCameraRenderThread::threadLoop`
  - `initializeImageHandler` creates a `VirtualCameraImageTransformingHandler`
    - `mEglDisplayContext` inits EGL/GLES context
    - `mEglTextureYuvProgram` draws input yuv texture to output yuv texture
      - it uses `GL_EXT_YUV_target` and `__samplerExternal2DY2YEXT` to sample
        and write raw yuv
    - `mEglSurfaceTexture` holds the input texture
      - when the owner app produces a frame, `requestTextureUpdate` is called
  - it handles `ProcessCaptureRequestTask` forever
    - when a client app calls `VirtualCameraSession::processCaptureRequest`, a
      `ProcessCaptureRequestTask` is enqueued
- `VirtualCameraRenderThread::processCaptureRequest`
  - `mImageHandler->waitForInputFrame` waits for the owner app to produce a frame
  - `mImageHandler->updateTexture`
    - `GLConsumer::updateTexImage` acquires the buffer, creates an EGLImage if
      needed, and binds to the texture
      - `doGLFenceWaitLocked` calls `eglWaitSyncKHR` to insert server wait
  - `renderOutputBuffers` calls `mImageHandler->fillOutputBuffer`
    - `renderIntoBlobStreamBuffer` if yuv to jpeg
    - `renderIntoImageStreamBuffer` if yuv to yuv
- `VirtualCameraImageTransformingHandler::renderIntoBlobStreamBuffer`
  - `mSessionContext.fetchOrCreateYuvFramebuffer` creates a temp
    `EglFrameBuffer` and allocs an intermediate ahb
  - `renderIntoEglFramebuffer` renders yuv to yuv on temp fb
  - `planesLock` locks the output buffer for cpu write
  - `compressJpeg` compresses the temp fb to the output buffer
- `VirtualCameraImageTransformingHandler::renderIntoImageStreamBuffer`
  - `mSessionContext.fetchOrCreateYuvFramebuffer` creates a temp
    `EglFrameBuffer` wrapping the output ahb
  - `renderIntoEglFramebuffer`
    - it waits for the output ahb fence
    - if there is no input buffer, `glCear`
    - else, `EglTestPatternProgram::draw` calls `glDrawElements`
    - `framebuffer.afterDraw` calls `glFinish`
