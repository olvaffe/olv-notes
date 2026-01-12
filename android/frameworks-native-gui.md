Android libgui
==============

## libgui interfaces

- `ISurfaceComposer` is an interface to SurfaceFlinger
  - `createConnection` creates a connection to SurfaceFlinger and returns
    `ISurfaceComposerClient`
  - `createGraphicBufferAlloc` creates an `IGraphicBufferAlloc`
  - `getCblk` returns the control block of SurfaceFlinger.  It has information
    such as the number of displays, the width, height, and orientation of a
    display,
  - `setTransactionState`
  - `freezeDisplay`
  - `unfreezeDisplay`
  - `setOrientation`
  - `bootFinished`
  - `captureScreen`
  - `turnElectronBeamOff`
  - `turnElectronBeamOn`
  - `authenticateSurface`
- `IGraphicBufferAlloc` is an interface for creating graphic buffers.
  - `createGraphicBuffer(w, h, format, usage, ...)` is used to create a
    `GraphicBuffer`
- `ISurfaceComposerClient`
  - `createSurface(params, name, display, w, h, format, flags)` is used to
    create an `ISurface`
  - `destroySurface` is used to destroy an `ISurface`
- `ISurface`
  - `getSurfaceTexture` returns the `ISurfaceTexture` of the Surface
- `ISurfaceTexture`
  - `requestBuffer(bufferIdx, buf)` returns the `GraphicBuffer` for the given
    index
  - `setBufferCount` sets the number of `GraphicBuffer`s
  - `dequeueBuffer(outBuf, w, h, format, usage)` dequeues a buffer and returns
    the index
  - `queueBuffer(buf, timestamp, outW, outH, outTransform)` queues a buffer for
    display
  - `cancelBuffer(buf)` cancels a dequeued buffer
  - `setCrop(rect)`
  - `setTransform(transform)`
  - `setScalingMode(mode)`
  - `query(what, outVal)`
  - `setSynchronousMode(enabled)` is set to true if waiting for vsync is
    desired.
  - `connect(api)`
  - `disconnect(api)`

## libgui objects

- `SurfaceComposerClient` is an object representing a connection to
  SurfaceFlinger
  - it wraps `ISurfaceComposerClient` and `ISurfaceComposer`
  - it has methods to create surfaces, set the size or position of surfaces, and
    etc.  Basically, anything composition related.
  - A surface created by this object is wrapped in `SurfaceControl`
- `SurfaceControl` is an object to control how a surface is composited
  - it is convenient object for using `SurfaceComposerClient`
  - it has `getSurface` method that reutrns a `Surface`
- `Surface` is an object representing a surface
  - it wraps `ISurface`
  - it is a `SurfaceTextureClient`
  - upon init, it initializes the texture usage to `USAGE_HW_RENDERER`
  - it has `lock` method that dequeues a buffer and map it for SW read/write
    - `setUsage` will be called to set the usage correctly
  - it has `unlockAndPost` meothd to unmap and queue the buffer
- `SurfaceTextureClient` is an `ANativeWindow`, the lowest level object
  representing a surface
  - it wraps `ISurfaceTexture`
- `SurfaceTexture` can be a server side or client side object
  - it uses `IGraphicBufferAlloc` to allocate buffers
  - `setBufferCountServer` sets the number of server buffers
  - `setBufferCount` sets the number of client buffers
- `GraphicBuffer` is an `android_native_buffer_t` (now called
  `ANativeWindowBuffer_t`)

