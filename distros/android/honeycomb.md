Android Honeycomb
=================

## libgui interfaces

* `ISurfaceComposer` is an interface to SurfaceFlinger
  * `createConnection` creates a connection to SurfaceFlinger and returns
    `ISurfaceComposerClient`
  * `createGraphicBufferAlloc` creates an `IGraphicBufferAlloc`
  * `getCblk` returns the control block of SurfaceFlinger.  It has information
    such as the number of displays, the width, height, and orientation of a
    display, 
  * `setTransactionState`
  * `freezeDisplay`
  * `unfreezeDisplay`
  * `setOrientation`
  * `bootFinished`
  * `captureScreen`
  * `turnElectronBeamOff`
  * `turnElectronBeamOn`
  * `authenticateSurface`
* `IGraphicBufferAlloc` is an interface for creating graphic buffers.
  * `createGraphicBuffer(w, h, format, usage, ...)` is used to create a
    `GraphicBuffer`
* `ISurfaceComposerClient`
  * `createSurface(params, name, display, w, h, format, flags)` is used to
    create an `ISurface`
  * `destroySurface` is used to destroy an `ISurface`
* `ISurface`
  * `getSurfaceTexture` returns the `ISurfaceTexture` of the Surface
* `ISurfaceTexture`
  * `requestBuffer(bufferIdx, buf)` returns the `GraphicBuffer` for the given
    index
  * `setBufferCount` sets the number of `GraphicBuffer`s
  * `dequeueBuffer(outBuf, w, h, format, usage)` dequeues a buffer and returns
    the index
  * `queueBuffer(buf, timestamp, outW, outH, outTransform)` queues a buffer for
    display
  * `cancelBuffer(buf)` cancels a dequeued buffer
  * `setCrop(rect)`
  * `setTransform(transform)`
  * `setScalingMode(mode)`
  * `query(what, outVal)`
  * `setSynchronousMode(enabled)` is set to true if waiting for vsync is
    desired.
  * `connect(api)`
  * `disconnect(api)`

## libgui objects

* `SurfaceComposerClient` is an object representing a connection to
  SurfaceFlinger
  * it wraps `ISurfaceComposerClient` and `ISurfaceComposer`
  * it has methods to create surfaces, set the size or position of surfaces, and
    etc.  Basically, anything composition related.
  * A surface created by this object is wrapped in `SurfaceControl`
* `SurfaceControl` is an object to control how a surface is composited
  * it is convenient object for using `SurfaceComposerClient`
  * it has `getSurface` method that reutrns a `Surface`
* `Surface` is an object representing a surface
  * it wraps `ISurface`
  * it is a `SurfaceTextureClient`
  * upon init, it initializes the texture usage to `USAGE_HW_RENDERER`
  * it has `lock` method that dequeues a buffer and map it for SW read/write
    * `setUsage` will be called to set the usage correctly
  * it has `unlockAndPost` meothd to unmap and queue the buffer
* `SurfaceTextureClient` is an `ANativeWindow`, the lowest level object
  representing a surface
  * it wraps `ISurfaceTexture`
* `SurfaceTexture` can be a server side or client side object
  * it uses `IGraphicBufferAlloc` to allocate buffers
  * `setBufferCountServer` sets the number of server buffers
  * `setBufferCount` sets the number of client buffers
* `GraphicBuffer` is an `android_native_buffer_t` (now called
  `ANativeWindowBuffer_t`)

## DisplayHardware

* when the fb returns true in `isUpdateOnDemand()`, `PARTIAL_UPDATES` is set.
  And the surface is made `EGL_BUFFER_DESTROYED`.  If it fails to set the
  surface to `EGL_BUFFER_DESTROYED`, `BUFFER_PRESERVED` is set.
* `SWAP_RECTANGLE` is set when `EGL_ANDROID_swap_rectangle` is supported for the
  surface.  But if `PARTIAL_UPDATES` is also set, it is preferred and
  `SWAP_RECTANGLE` is cleared.
* To flip the surface, if `SWAP_RECTANGLE` is set, the swap rectangle is first
  set to the bounding rect of the dirty region.
  * If `PARTIAL_UPDATES` is set instead, `setUpdateRectangle()` is called.
  * Finally, `eglSwapBuffers` is called.

## SurfaceFlinger Texture Manager (not used?)

* `initTexture` initializes a `Texture` or `Image`
  * it calls `glGenTextures`
  * it sets the texture parameters
  * when there is an `Image`, it will decide whether to use `GL_TEXTURE_2D` or
    `GL_TEXTURE_EXTERNAL_OES` as the target
* An `Image` is initialized by `initEglImage`
  * it is initialized from a `GraphicBuffer`
  * it has an texture object and a `EGLImageKHR`
  * its texture image is set using `glEGLImageTargetTexture2DOES`
* A `Texture` is initialized by `loadTexture`
  * it is initialized from a `GGLSurface`
  * its texture image is set using `glTexImage2D`

## SurfaceFlinger objects

* `LayerBase` implements `ISurface`
* `Layer` is a `LayerBase` with `SurfaceTextureLayer`
* `SurfaceTexture` (it may also be used by a client)
  * `mQueue` is a queue of indices of the buffers that are to be displayed.  In
    sync mode, every buffer will be displayed in order.  In async mode, the
    queue has at most one index.  When a new buffer is queued before the current
    one is displayed, the new one replaces the current one.
  * A buffer is initially `FREE`.  After calling `dequeueBuffer`, it becomes
    `DEQUEUED`.  After calling `enqueueBuffer`, it becomes `QUEUED`.  When
    `updateTexImage` is called, the buffer is made current (streamed to the
    texture object) and is erased from `mQueue`.  However, it remains `QUEUED`
    until next `updateTexImage` call.  This makes sure the client does not see
    the buffer that is still in use.
    * In sync mode, a current buffer can be dequeued when it is free.  But that
      is never the case.  The doc is outdated?
  * Sync mode: `dequeueBuffer` blocks until there is a slot available.  Buffers
    queued in sync mode will be retired in order.
    * That is, every frame is shown.
  * Async mode: `dequeueBuffer` never blocks.  It returns error if there is no
    slot available.  There is at most one queued buffer and queueing another
    buffer will replace the current one.
    * That is, there can be frame drops
  * `updateTexImage` streams the first buffer in the queue to the texture object
    and clears it from the queue.  The state of the buffer is still `QUEUED`
    until next `updateTexImage` call
  * A video decoder may process multiple frames parellelly.  It tends to dequeue
    multiple buffers at the same time.
  * `MIN_ASYNC_BUFFER_SLOTS` is 3
  * `MIN_SYNC_BUFFER_SLOTS` is 2
  * client buffer count is used only when it is not zero.
  * at most one buffer can be dequeued if client buffer count is not set
