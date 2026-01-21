Android SurfaceFlinger
======================

## Native Activity

- before looking into how surfaceflinger works, let's look into how a native
  activity interacts with sf
  - upon `onNativeWindowCreated`, app receives an `ANativeWindow`
  - `ANativeWindow_lock` locks the win for cpu access
  - `ANativeWindow_unlockAndPost` unlocks and presents the updated contents
  - egl and vk also accept `ANativeWindow` for gpu access
- internally, when an activity is created,
  - a `SurfaceControl` is created to connect to the sf
  - it also creates a surface for drawing, at which point
    `onNativeWindowCreated` is called
- inside libgui,
  - a `SurfaceControl` owns
    - a `SurfaceComposerClient` which is the connection to sf
    - a `SurfaceControl` which is a child surface created from sf
    - a `BLASTBufferQueue` that wraps the child surface
    - a `Surface` that wraps the blast bq and is a `ANativeWindow`
  - `ANativeWindow_lock` calls `Surface::lock` to dequeue a buffer from
    `BBQBufferQueueProducer` and locks it for cpu access
    - the buffer is either reused or newly allocated directly
  - `ANativeWindow_unlockAndPost` calls `Surface::unlockAndPost` to unlocks
    the buffer and queues it to `BBQBufferQueueProducer`
    - `SurfaceComposerClient::Transaction::setBuffer` adds buffer change to
      the transaction
- on the sf side,
  - `SurfaceComposerAIDL::createConnection` creates a `Client`
  - `Client::createSurface` creates a `Layer`
  - `SurfaceFlinger::setTransactionState` handles a batch of transactions
    - client batches their transactions
    - this function queues up transactions and notifies the scheduler
  - `SurfaceFlinger::commit` commits transactions
    - this is called from the scheduler
    - `SurfaceFlinger::updateLayerSnapshots` applies the transactions
      - `Layer::setBuffer` handles buffer change
  - `SurfaceFlinger::composite` composites a frame
    - this is also called from the scheduler
    - `CompositionEngine::present` composites and presents
      - this calls `Output::present` of each output
        - `Output::presentFrameAndReleaseLayers`
        - `Display::presentFrame`
          - note that a `Display` is a `Output`
        - `HWComposer::presentAndGetReleaseFences`
        - `HWC2::impl::Display::present`

## GPU Composition

- `adb shell service call SurfaceFlinger 1008 i32 1` forces gpu composition
- during init, `SurfaceFlinger::processDisplayAdded` adds a `DisplayDevice`
  - it creates a `LegacyFramebufferSurface` which is like a bq
  - it creates a `DisplayDevice` over the legacy fb surface
    - `Display::createRenderSurface` to create a `RenderSurface` where
      - `mNativeWindow` is the producer end of the fb
      - `mDisplaySurface` is the consumer end of the fb
- `Output::present`
  - `Output::dequeueRenderBuffer` dequeues a buf from fb
  - `Output::composeSurfaces` performs gpu composition to the buf
    - `RenderEngine::drawLayers` draws
  - `RenderSurface::queueBuffer` queues the buf
  - `LegacyFramebufferSurface::advanceFrame` calls
    `HWComposer::setClientTarget`

## Protocol Overview

- `ISurfaceComposer` is the interface of `SurfaceFlinger`
  - `ISurfaceComposerClient` is the interface of a connection to
    `SurfaceFlinger`.  It can be created from `ISurfaceComposer`
    - `ISurface` is the interface of a on-screen surface.  It can be created
      from `ISurfaceComposerClient`.
      - `ISurfaceTexture` is the interface of the buffer storage of the
        surface.  It can be obtained from `ISurface`.
  - `IGraphicBufferAlloc` is the interface of a buffer allocator.  It can be
    created from `ISurfaceComposer`.  Buffers are allocated from this
    interface.
- This is how things work
  - an app creates `ISurfaceComposer` to talk to `SurfaceFlinger`.  It
    creates a connection, that is, `ISurfaceComposerClient`.  `ISurface`s can
    be created from the connection.  `ISurfaceTexture`s can be used to access
    surface buffers.
  - the app can also use `IGraphicBufferAlloc` to allocate offscreen buffers.

## Object Overview

- `SurfaceComposerClient` wraps `ISurfaceComposer` and
  `ISurfaceComposerClient`.
- `SurfaceControl` wraps `ISurface`.
- `Surface` wraps `SurfaceTextureClient` which wraps `ISurfaceTexture`
  - `Surface` is an old object, which becomes very thin after the introduction
    of `SurfaceTextureClient`
- `SurfaceTexture` wraps `ISurfaceCompoer` and `IGraphicBufferAlloc`.
  - It is used by clients to allocate offscreen buffers.  EGL can render to
    them, by wrapping `SurfaceTexture` in a `SurfaceTextureClient`.  Or sample
    from them, by using `EGLImage`.
  - It is also used by the server to allocate buffers and implement
    `ISurfaceTexture`
  - Because it implements `ISurfaceTexture`, it can be wrapped in
    `SurfaceTextureClient`

## EGL native types

- `FramebufferNativeWindow` is an `ANativeWindow`.  It is server-only.
- `SurfaceTextureClient` is an `ANativeWindow`.  It is client-only.
  - Thus, `Surface` and `SurfaceTexture` can both be viewed as native types
- `GraphicBuffer` is an `ANativeWindowBuffer`.  It is used by both clients and
  the server.

## Jave-level objects

- Java-level `ViewRootImpl` has a Java-level `Surface`, which maps to
  C++-level `Surface`.
- Java-level `SurfaceView` also has a Java-level `Surface`.
- Java-level `TextureView` has a Java-level `SurfaceTexture`, which maps to
  C++-level `SurfaceTexture`.

## `DisplaySurface`

- has two implementations: `FramebufferSurface` and `VirtualDisplaySurface`
  - a `FramebufferSurface` is created for each internal or externail physical
    display
  - a `VirtualDisplaySurface` is created for each virtual display
- `beginFrame` is called when SF is about to render a new frame
- `prepareFrame` is called after SF knows whether client composition is
  necessary 
- `compositionComplete` is called after all layers are rendered (but before SF calls `eglSwapBuffers`)
- `advanceFrame` is called after all layers are rendered (and after SF calls `eglSwapBuffers`)
- `onFrameCommitted` is called after the new frame is presented/posted
- `FramebufferSurface` is implemented on top of a `IGraphicBufferConsumer`
  - `beginFrame`, `prepareFrame`, and `compositionComplete` are no-ops
  - `advanceFrame` acquires the next buffer in BQ and makes it the client
    target in HWC.  If the buffer is different from the previous one, the
    previous one is marked as pending release. 
  - `onFrameCommitted` releases the previous buffer.  It gets the present
    fence from HWC and associates the fence with the previous buffer.  This
    way, the previous buffer is not reused until the fence is signaled.
  - this requires `SurfaceFlinger::maxFrameBufferAcquiredBuffers` to be at
    least 2

## `DisplayDevice`

- a `DisplayDevice` is created for each display
  - it is created with an HWC display, a `DisplaySurface`, an IGBP, and an
    optinal `EGLConfig` (no config when `EGL_KHR_no_config_context` is
    supported)
  - it will create a `Surface` from the IGBP, and an `EGLSurface` from the
    surface.

## `EGLImage` for client target

- `DisplayDevice` owns N buffers and create an `EGLImage` for each of the
  buffer.
- It communicates with `DisplaySurface` which image is current

## BQ Fencing

- If supported, a producer can queue a buffer with a fence
  - gralloc unlock may return a fence
  - `eglSwapBuffers` may queue the buffer with a fence
  - `vkQueuePresentKHR` may queue the buffer with a fence
- the consumer will acquire the buffer with the associated fence
  - it will give the fence to HWC along with the buffer
  - it will also bind the buffer to a GL texture.  It needs to perform a
    GPU `eglWaitSyncKHR` or CPU `sync_wait` on the fence before the GL texture
    can be used
- the consumer can release the buffer with a different fence as well
  - HWC may return a fence for the buffer
  - GLES composition may have a fence for the client target
  - Additionally, `GLConsumer` needs to, dpending on driver support,
    - create another fence for the outstanding GL commands, or
    - create EGLSync for the outstanding GL commands
- when the producer dequeue the released buffer again
  - the consumer release fence will be returned with the buffer
  - if there is an EGLSync outstanding, dequeue needs to block until
    `eglClientWaitSyncKHR` returns

## dumpsys

- `adb shell dumpsys SurfaceFlinger`
- `adb shell dumpsys SurfaceFlinger --timestats -enable -clear`
  - this enables and clears timestats, for fps and etc.
  - one can periodically `-dump -clear` to dump and clear the stats

## Frame Trace View 

- this is Unity on ARCVM
- frame pacing depends on vsyncs
  - `IRQ (arc_timer)` fires every 16.6ms
  - SF `TimerDispatch` thread wakes up, which wakes up `app` thread and
    `sf`thread
    - it also flips `VSYNC-app` and `VSYNC-sf` events
  - SF `app` thread wakes up APP `UI thread` to `Choreographer#doFrame`
    - for Unity, it also wakes up APP `UnityChoreograp` to
      `Choreographer#doFrame`
- rendering depends on fences
  - APP `UnityGfxDeviceW` thread calls `dequeueBuffer` and waits until the
    buffer is idle
  - It then renders to the buffer and does a `eglSwapBuffers` to `queueBuffer`
  - If the buffer is idle by next vsync, SF latches it
  - when systrace is enabled, APP `HWC release` and `GPU completion` threads
    are created
    - APP `HWC release` tracks all out-fences returned in `dequeueBuffer`
    - APP `GPU completion` tracks all in-fences upon `queueBuffer`
- at perfectly stable 60fps,
  - the three buffers of a buffer queue are
    - one is owned by HWC
    - one is owned by SF to be latched
    - one is in the buffer queue
  - on vsync,
    - SF latches and sends the buffer it owns to HWC
    - the one owned by HWC is returned to the buffer queue
    - app dequeues from the queue to render the next frame and passes it to SF
- a GPU-bound app
- a CPU-bound app

## Vulkan

- Android vulkan loader implements `VK_KHR_android_surface` and `VK_KHR_swapchain`
- `vkCreateAndroidSurfaceKHR`
  - it creates a `VkSurfaceKHR` from a `ANativeWindow`, which is from
    `ANativeWindow_fromSurface`
  - internally, the concrete type is `Surface` which is a wrapper over
    `IGraphicBufferProducer`
- `vkQueuePresentKHR` calls `ANativeWindow::queueBuffer`
  - it calls `Surface::queueBuffer` which calls
    `BpGraphicBufferProducer::queueBuffer`
    - `BpGraphicBufferProducer` is a proxy object in the app process
    - `BufferQueueProducer` is the real object in the sf process
      - `BufferQueueProducer` implements `BnBufferQueueProducer`
      - `BnBufferQueueProducer::onTransact` calls
      	`BufferQueueProducer::queueBuffer`
  - `BufferQueueProducer::queueBuffer` adds the buffer to the queue
    - it has a `waitForever` on the previous buffer to throttle
  - when tracing is on, after `BufferQueueProducer::queueBuffer` returns,
    `Surface::queueBuffer` adds the fence to `gpuCompletionThread` in the app
    process
    - there is a `GPU completion` thread that waits on the fence
    - the thread has a `GPU completion` counter that tracks the number of
      fences pending in the thread
- `vkAcquireNextImageKHR` calls `ANativeWindow::dequeueBuffer`
  - similarly, it calls `BufferQueueProducer::dequeueBuffer`

## Graphics

- android.graphics: Skia
- android.opengl: utils
- javax.microedition.khronos.egl: EGL
- javax.microedition.khronos.opengles: OpenGL ES
- Skia: libcorecg, libsgl, libskiagl
- OpenGL ES: `libGLESv1_CM.so`, stubs call into hooks
- EGL: libEGL.so, dlopen libhgl.so or libagl.so to get hooks and call into hooks
- libhgl.so or libagl.so: Real libraries defining gl and egl functions
- 

    typedef struct egl_native_window_t*     EGLNativeWindowType;
    typedef struct egl_native_pixmap_t*     EGLNativePixmapType;
    typedef void*                           EGLNativeDisplayType;

memory management:
- IMemory.h MemoryHeapBase.h MemoryBase.h MemoryDealer.h MemoryHeapPmem.h
- IMemoryHeap: a (fd, size, flags) tuplet, abstraction of mmap, without mmap offset!
- IMemory: complete IMemoryHeap with offset.  say the underlying heap is (FD, SIZE, FLAGS).
           a IMemory may have offset SIZE/2 and size SIZE/2, indexing the second half of the heap.
- class MemoryHeapBase : public virtual BnMemoryHeap: trivial impl., supporting ashmem, fd, path
- class MemoryBase : public BnMemory: trivial impl by wrapping a sp<IMemoryHeap>.
- MemoryDealer: dealer of a heap, backed by HeapInterface and AllocatorInterface.  Note that they does not implement Bp nor Bn for RPC.
  SharedHeap: inherits HeapInterface and MemoryHeapBase.  takes (offset, size) and returns IMemory.

              /* HeapInterface:
               - interface for implementing a "heap". A heap basically provides
               - the IMemoryHeap interface for cross-process sharing and the
               - ability to map/unmap pages within the heap.
               -/
  SimpleBestFitAllocator: inherits AllocatorInterface. impl. an algorithm to find the best (offset, size) pair given a total size.

window manager service <-> surfaceflinger
- window manager service asks surfaceflinger to create surfaces
- how does wm service (remote) get a sp<Surface>?

    sp<ISurfaceComposer>: remote SurfaceComposerClient asks service manager for "SurfaceFlinger" service which inherits BnSurfaceComposer
                          Use interface_cast to get BpSurfaceComposer.
    sp<ISurfaceFlingerClient>: sp<ISurfaceComposer>->createConnection -> Bp -> Bn -> SurfaceFlinger::createConnection.
  			     a BClient is created which inherits BnSurfaceFlingerClient.
  			     remote interface_cast it to get BpSurfaceFlingerClient.
    sp<ISurface>: sp<ISurfaceFlingerClient>::createSurface -> Bp -> Bn -> BClient::createSurface -> SurfaceFlinger::createSurface.
  		one of Layer{,Blur,Dim} is created.  It inherits LayerBaseClient.
  		LayerBaseClient::getSurface method returns sp<LayerBaseClient::Surface>, which inherits BnSurface.
    sp<Surface>: remote goes further and wraps sp<ISurface> in Surface in its address space.  (which is a member of android.view.Surface)
- remote openTransaction -> make changes to surfaces -> closeTransaction (and commit to surfaceflinger through `SET_STATE`)

SurfaceFlinger:
- init(), thread created and calls readyToRun(), and enter threadLoop()
- mTransactionCV: transaction on one thread, wait for handling by another thread
  mSyncObject: signal drawing thread an new event (tranaction or others)
- mServerHeap: 4K ashmem, all used by mServerCblkMemory, which is casted to `surface_flinger_cblk_t *mServerCblk`
  control block has infomations like which display is connected (only 1 for now), display's properties.
- mCPU = GPUFactory::getGPU();
- `mSurfaceHeapManager = new SurfaceHeapManager(this, 8 << 20);`
  surface heap manager returns a memory dealer of 8MB for each client.
  the returned dealer might be from GPU, from PMEM, or plain MemoryDealer from ashmem
  The pmem driver is used to manage large (1-16+MB) physically contiguous regions of memory shared between userspace and kernel drivers (dsp, gpu, etc), mainly for MSM7201A
- a DisplayHardware is created, which has EGLDisplaySurface (with grandparent `egl_native_window_t`, namely, /dev/fb0).
  a EGLSurface is created around mDisplaySurface.
- Client: mCblkHeap: 4K ashmem, all used by mCblkMemory.  `per_client_cblk_t`
  contains client states and all surfaces's info; a mSharedHeapAllocator is
  allocated (8MB) which is used normally by all surfaces of the client
- setClientState: change the states of client surfaces, setting transaction flags
- When a surface is busy, locking the surface requires waiting.
  `scheduleBroadcast` is used on the server side to wake waiting clients up.

SurfaceFlinger::createSurface
- Layer{,Blur,Dim,Buffer} inherits LayerBaseClient; LayerOrientationAnim inherits LayerBase; LayerBitmap: use by all mentioned child classes
- LayerBitmap is init()'ed with a MemoryDealer.  It is where the real buffer is
  allocated.  It provides a high-level view of the buffer, which buffer alignment,
  bitmap resizing, and graphics operations using GGLSurface.  `surface_info_t`
  found in `layer_cblk_t` is also initialized by bitmap.
- LayerBase stores the context (op, states) of the surface.
- Layer creates two allocators (double buffering), used by two LayerBitmaps.  A
  LayerBaseClient::Surface inheriting BnSurface is created with the two
  bitmaps' heaps, which is quite dumb though.  The two heaps are returned
  to remote on createSurface, which makes graphics op possible on remote
  (possible because of heaps, together with `surface_info_t->bits_offset`)
- surfaceflinger creates a Layer and `addLayer_l`, calls layer's setBuffers,
  initStates, getSurface and getSurfaceData.
- every layerbase is given a id, from sIdentity++
- secure means zero the memory

## Layer

- A layer has `mVertices`.  It is layer's geometry converted to 16.16 fixed point.
  Its values are

    mVertices[0] = (x, y)
    mVertices[1] = (x, y+h)
    mVertices[2] = (x+w, y+h)
    mVertices[3] = (x+w, y)
  where `w` and `h` are positive.  It uses X-window coordinates.
- In `LayerBase::drawWithOpenGL`, `mVertices` is used directly as vertex array.
  Since OpenGL uses cartesian coordinates, the result is the back face with
  upside down image.  With an upside down projection, it gives the wanted result.

SurfaceLinger::threadLoop
- two stage: setClientState changes states; handleTransaction applies states.
  When a global transaction is in action, handleTransaction is not called.
  (Java Surface always does global transaction, see `Surface_openTransaction`)
- handleTransaction: if a connection/surface is made/destroyed, or orientation
  changed, or etc., eTransactionNeeded is set.  if a layer position changed,
  size changed, etc. eTraversalNeeded is set.  All layers marked is
  doTransaction.  surfacelinger then processes itself, and copy mCurrentState to
  mDrawingState.
- handlePageFlip: lockPageFlip all layers, compute mDirtyRegion, unlockPageFlip
  all layers.  flips the front/back buffers.  later after repaint,
  finishPageFlip is called.  By flipping, it changes the front/back indices, as
  can be seen in Layer::post.  When the client locks the surface next time, front
  is memory copied to back.
- what remote draws is wrapped in GGLSurface (LayerBitmap) and turned into
  texture through LayerBase::loadTexture.
- handleRepaint: calls each layer->draw
- hw.flip: eglSwapBuffers

EGLDisplaySurface:
- mapFrameBuffer maps /dev/fb0 and wrap it in GGLSurface.  If fb0 supports
  flipping, mFb[1] points to the other half, otherwise, mFb[1] uses malloc()
- no MemoryDealer -> no one, except itself, directly draws to fb
- mBlitEngine is set if copybit.so is available
- `egl_native_window_t = { .base = intptr_t(mFb[0].data), .offset = intptr_t(mFb[1 - mIndex].data) - egl_native_window_t::base }`
  i.e., native window used for drawing is based by back buffer.
- on swapBuffer, back buffer is copied to front buffer

DisplayHardware:
- DisplayHardwareBase: console management (text/graphics mode, switching to vt7, etc. if not on msm7k?)
			switching VT -> acquire/release screen
- on init, a new EGLDisplaySurface is allocated, and a EGLSurface (window surface) is created.
- on flip, eglSwapBuffer
- flags
  - `SWAP_RECTANGLE_EXTENSION` and `UPDATE_ON_DEMAND` are always set
  - `SLOW_CONFIG` if `EGL_CONFIG_CAVEAT`
  - `BUFFER_PRESERVED` if `EGL_BUFFER_PRESERVED` is set in `EGL_SWAP_BEHAVIOR`
  - `NPOT_EXTENSION` if `GL_ARB_texture_non_power_of_two`
  - `DRAW_TEXTURE_EXTENSION` if `GL_OES_draw_texture`
  - `DIRECT_TEXTURE` if `GL_ANDROID_direct_texture`
- When `relsig` is received in `DisplayHardwareBase`, `screenReleased` is called
  to queue `eConsoleReleased`.  When the event is processed, it gives
  `DisplayHardware` to chance to do anything it wants before ACK.
- Similar to `acqsig`

## `per_client_cblk_t`

- `per_client_cblk_t` has

        struct per_client_cblk_t   // 4KB max
        {
                        Mutex           lock;
                        Condition       cv;
                        layer_cblk_t    layers[NUM_LAYERS_MAX] __attribute__((aligned(32)));

            enum {
                BLOCKING = 0x00000001,
                INSPECT  = 0x00000002
            };

            per_client_cblk_t();

            // these functions are used by the clients
            status_t validate(size_t i) const;
            int32_t lock_layer(size_t i, uint32_t flags);
            uint32_t unlock_layer_and_post(size_t i);
            void unlock_layer(size_t i);
        };
- `(state & eIndex)` gives the buffer the client can use (back buffer).  It is
  complicated by `eFlipRequested`, `eNextFlipPending`, and `eBusy`.
  If some (comibnatin of) the flags are set, the client should wait before it
  can start using.
- `lock_layer` waits until one buffer is available, locks the specified layer by
  setting `eLocked` flag and returns back buffer (0 or 1)
- the mutex is only used for waiting on condition (cv.wait)
- `unlock_layer` unsets `eLocked` flag
- `unlock_layer_and_pos` sets `eFlipRequested` and/or `eNextFlipPending` and
  unsets `eLocked`
- `eLocked` indicates that the client is using it.  It prevents resize to happen
  on the server side, instead, the request is rescheduled by setting
  `eRestartTransaction`
- `eBusy` indicates that the server is flipping (between `lockPageFlip` and
  `finishPageFlip`).

## SurfaceComposerClient

- There are 4 types of CBLK: `surface_flinger_cblk_t`, `display_cblk_t`,
  `per_client_cblk_t`, and `layer_cblk_t`
- `lockSurface` locks and returns the info of the back buffer.  Current dirty
  region is copied from front buffer to back buffer, and is updated to new one.
- `unlockAndPostSurface` sends current dirty region to server (by storing in
  `layer_cblk_t`), calls `unlock_layer_and_post` and signal server.  If there is
  swap rectangle, it is sent instead of dirty region.
- The rest deals with states, which is sent to server through `setState`

## Surface

- For 2D usage, `Surface` wraps the underlying buffer in a `SkBitmap` and
  operates on a `SkCanvas`.  See `Surface_lockCanvas`.
- For 3D usage, `Surface` wraps the underlying buffer in a
  `EGLNativeWindowSurface` and operates in a `GL` context.  The
  `EGLNativeWindowSurface` is passed to `eglCreateWindowSurface` and the
  returned `EGLSurface` is made current in `EGL`.  See `eglCreateWindowSurface`
  in `EGLImpl.java`.

## Dirty Region

- `Layer` has `mPostedDirtyRegion`, as sent by client when `unlockAndPost`
- `LayerBase` has
  - `visibleRegionScreen`, the region which is visible by the user
  - `transparentRegionScreen`, the region hinted by client which should be
    considered transparent, i.e., should be clipped out
  - `coveredRegionScreen`, the region covered by upper layers
- `transparentRegionScreen` is not part of `visibleRegionScreen`
- `SurfaceFlinger`'s regions
  - `mDirtyRegion`: the dirty region decided in `handlePageFlip`.
    It is added the union of all layers' dirty regions
  - `mWormholeRegion`: screen region minus opaque region.  That is, the region
    not covered by any layer.  It is usually a bug if non-empty.  Painted black.
  - `mInvalidRegion`: the region used by DisplayHardware::flip to perform
                  copy-back optimization.  (copy-back: instead of copy back
                  buffer to front before eglSwapBuffer, switch back and front,
                  and copy from back to front?)
- `DisplayHardware` (`EGLDisplaySurface` actually) is double-buffered to avoid
  flicker.  `SurfaceFlinger`'s layers are double-buffered to avoid lock
  contension between server and client.
- computeVisibleRegions: compute and set new visible region, rarely called.
  A layer has visible bounds.  If layer is hidden or alpha==0, visible region
  is empty.  Otherwise, visible region is visible bounds minus transparent
  region (the region that is hinted not to be drawn) minus aboveOpaqueLayers.
  Covered region is visible bounds intersected with aboveCoveredLayers.
  If layer is opaque, opaque region is visible region.
  Now, if layer is dirty, changed region is union of (previous visible region)
  and (current visible region).  If layer is not dirty, changed region is
  intersection of (current visible region) and (previous covered region).
  The changed region minus aboveOpaqueLayers is the possible region to flip,
  which will later be modified.

## Effect hooks

- When a client sets the states, the new states are remembered in
  `mCurrentState` of a layer and the surfaceflinger is signaled.
- The main thread calls `handleTransaction` to copy `mCurrentState` to
  `mDrawingState`.
- It then calls `handlePageFlip` to handle front/back buffers swapping.
- Finally, `composeSurfaces` are called to draw.
- There should be hooks for
  - input event
  - state change
  - buffer swapping not needed because it is implied by drawing
  - create/destroy of layers
  - `composeSurfaces`
  - layer drawing
- Conversely, when an effect wants to schedule a work, it calls `signalEvent`
- how to draw a layer?
  - update geometry and draw geometry with window texture
  - `uploadGeometry`
    - index buffer and vertex buffer
  - `drawGeometry`
    - with layer state
    - with user transform and attrib
  - `draw` calls both
- `drawWithOpenGL`
  - framebuffer has no alpha channel; layers use pre-multiplied alpla
  - `needsBlending` returns true if the pixel format has alpha channel
  - 

## DisplayHardware

- when the fb returns true in `isUpdateOnDemand()`, `PARTIAL_UPDATES` is set.
  And the surface is made `EGL_BUFFER_DESTROYED`.  If it fails to set the
  surface to `EGL_BUFFER_DESTROYED`, `BUFFER_PRESERVED` is set.
- `SWAP_RECTANGLE` is set when `EGL_ANDROID_swap_rectangle` is supported for the
  surface.  But if `PARTIAL_UPDATES` is also set, it is preferred and
  `SWAP_RECTANGLE` is cleared.
- To flip the surface, if `SWAP_RECTANGLE` is set, the swap rectangle is first
  set to the bounding rect of the dirty region.
  - If `PARTIAL_UPDATES` is set instead, `setUpdateRectangle()` is called.
  - Finally, `eglSwapBuffers` is called.

## SurfaceFlinger Texture Manager (not used?)

- `initTexture` initializes a `Texture` or `Image`
  - it calls `glGenTextures`
  - it sets the texture parameters
  - when there is an `Image`, it will decide whether to use `GL_TEXTURE_2D` or
    `GL_TEXTURE_EXTERNAL_OES` as the target
- An `Image` is initialized by `initEglImage`
  - it is initialized from a `GraphicBuffer`
  - it has an texture object and a `EGLImageKHR`
  - its texture image is set using `glEGLImageTargetTexture2DOES`
- A `Texture` is initialized by `loadTexture`
  - it is initialized from a `GGLSurface`
  - its texture image is set using `glTexImage2D`

## SurfaceFlinger objects

- `LayerBase` implements `ISurface`
- `Layer` is a `LayerBase` with `SurfaceTextureLayer`
- `SurfaceTexture` (it may also be used by a client)
  - `mQueue` is a queue of indices of the buffers that are to be displayed.  In
    sync mode, every buffer will be displayed in order.  In async mode, the
    queue has at most one index.  When a new buffer is queued before the current
    one is displayed, the new one replaces the current one.
  - A buffer is initially `FREE`.  After calling `dequeueBuffer`, it becomes
    `DEQUEUED`.  After calling `enqueueBuffer`, it becomes `QUEUED`.  When
    `updateTexImage` is called, the buffer is made current (streamed to the
    texture object) and is erased from `mQueue`.  However, it remains `QUEUED`
    until next `updateTexImage` call.  This makes sure the client does not see
    the buffer that is still in use.
    - In sync mode, a current buffer can be dequeued when it is free.  But that
      is never the case.  The doc is outdated?
  - Sync mode: `dequeueBuffer` blocks until there is a slot available.  Buffers
    queued in sync mode will be retired in order.
    - That is, every frame is shown.
  - Async mode: `dequeueBuffer` never blocks.  It returns error if there is no
    slot available.  There is at most one queued buffer and queueing another
    buffer will replace the current one.
    - That is, there can be frame drops
  - `updateTexImage` streams the first buffer in the queue to the texture object
    and clears it from the queue.  The state of the buffer is still `QUEUED`
    until next `updateTexImage` call
  - A video decoder may process multiple frames parellelly.  It tends to dequeue
    multiple buffers at the same time.
  - `MIN_ASYNC_BUFFER_SLOTS` is 3
  - `MIN_SYNC_BUFFER_SLOTS` is 2
  - client buffer count is used only when it is not zero.
  - at most one buffer can be dequeued if client buffer count is not set

## `captureScreen`

- `renderScreenImplLocked` draws all layers from bottom to top
  - `layersSortedByZ` is from bottom to top
  - when an activity has a surface view, the layer of the surface view is
    under the layer of the activity.  A hole is dug out from the layer of the
    activity
    - `transparentRegionHint` is just a hint
    - I think it clears the alpha channel to 0.0

## Client Composition

- U defaults to `SKIA_GL_THREADED`
  - `debug.renderengine.backend` can override
