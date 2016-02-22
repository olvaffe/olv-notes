Graphics
========

* android.graphics: Skia
* android.opengl: utils
* javax.microedition.khronos.egl: EGL
* javax.microedition.khronos.opengles: OpenGL ES
* Skia: libcorecg, libsgl, libskiagl
* OpenGL ES: `libGLESv1_CM.so`, stubs call into hooks
* EGL: libEGL.so, dlopen libhgl.so or libagl.so to get hooks and call into hooks
* libhgl.so or libagl.so: Real libraries defining gl and egl functions
* 

    typedef struct egl_native_window_t*     EGLNativeWindowType;
    typedef struct egl_native_pixmap_t*     EGLNativePixmapType;
    typedef void*                           EGLNativeDisplayType;

memory management:
* IMemory.h MemoryHeapBase.h MemoryBase.h MemoryDealer.h MemoryHeapPmem.h
* IMemoryHeap: a (fd, size, flags) tuplet, abstraction of mmap, without mmap offset!
* IMemory: complete IMemoryHeap with offset.  say the underlying heap is (FD, SIZE, FLAGS).
           a IMemory may have offset SIZE/2 and size SIZE/2, indexing the second half of the heap.
* class MemoryHeapBase : public virtual BnMemoryHeap: trivial impl., supporting ashmem, fd, path
* class MemoryBase : public BnMemory: trivial impl by wrapping a sp<IMemoryHeap>.
* MemoryDealer: dealer of a heap, backed by HeapInterface and AllocatorInterface.  Note that they does not implement Bp nor Bn for RPC.
  SharedHeap: inherits HeapInterface and MemoryHeapBase.  takes (offset, size) and returns IMemory.

              /* HeapInterface:
               * interface for implementing a "heap". A heap basically provides
               * the IMemoryHeap interface for cross-process sharing and the
               * ability to map/unmap pages within the heap.
               */
  SimpleBestFitAllocator: inherits AllocatorInterface. impl. an algorithm to find the best (offset, size) pair given a total size.

window manager service <-> surfaceflinger
* window manager service asks surfaceflinger to create surfaces
* how does wm service (remote) get a sp<Surface>?

    sp<ISurfaceComposer>: remote SurfaceComposerClient asks service manager for "SurfaceFlinger" service which inherits BnSurfaceComposer
                          Use interface_cast to get BpSurfaceComposer.
    sp<ISurfaceFlingerClient>: sp<ISurfaceComposer>->createConnection -> Bp -> Bn -> SurfaceFlinger::createConnection.
  			     a BClient is created which inherits BnSurfaceFlingerClient.
  			     remote interface_cast it to get BpSurfaceFlingerClient.
    sp<ISurface>: sp<ISurfaceFlingerClient>::createSurface -> Bp -> Bn -> BClient::createSurface -> SurfaceFlinger::createSurface.
  		one of Layer{,Blur,Dim} is created.  It inherits LayerBaseClient.
  		LayerBaseClient::getSurface method returns sp<LayerBaseClient::Surface>, which inherits BnSurface.
    sp<Surface>: remote goes further and wraps sp<ISurface> in Surface in its address space.  (which is a member of android.view.Surface)
* remote openTransaction -> make changes to surfaces -> closeTransaction (and commit to surfaceflinger through `SET_STATE`)

SurfaceFlinger:
* init(), thread created and calls readyToRun(), and enter threadLoop()
* mTransactionCV: transaction on one thread, wait for handling by another thread
  mSyncObject: signal drawing thread an new event (tranaction or others)
* mServerHeap: 4K ashmem, all used by mServerCblkMemory, which is casted to `surface_flinger_cblk_t *mServerCblk`
  control block has infomations like which display is connected (only 1 for now), display's properties.
* mCPU = GPUFactory::getGPU();
* `mSurfaceHeapManager = new SurfaceHeapManager(this, 8 << 20);`
  surface heap manager returns a memory dealer of 8MB for each client.
  the returned dealer might be from GPU, from PMEM, or plain MemoryDealer from ashmem
  The pmem driver is used to manage large (1-16+MB) physically contiguous regions of memory shared between userspace and kernel drivers (dsp, gpu, etc), mainly for MSM7201A
* a DisplayHardware is created, which has EGLDisplaySurface (with grandparent `egl_native_window_t`, namely, /dev/fb0).
  a EGLSurface is created around mDisplaySurface.
* Client: mCblkHeap: 4K ashmem, all used by mCblkMemory.  `per_client_cblk_t`
  contains client states and all surfaces's info; a mSharedHeapAllocator is
  allocated (8MB) which is used normally by all surfaces of the client
* setClientState: change the states of client surfaces, setting transaction flags
* When a surface is busy, locking the surface requires waiting.
  `scheduleBroadcast` is used on the server side to wake waiting clients up.

SurfaceFlinger::createSurface
* Layer{,Blur,Dim,Buffer} inherits LayerBaseClient; LayerOrientationAnim inherits LayerBase; LayerBitmap: use by all mentioned child classes
* LayerBitmap is init()'ed with a MemoryDealer.  It is where the real buffer is
  allocated.  It provides a high-level view of the buffer, which buffer alignment,
  bitmap resizing, and graphics operations using GGLSurface.  `surface_info_t`
  found in `layer_cblk_t` is also initialized by bitmap.
* LayerBase stores the context (op, states) of the surface.
* Layer creates two allocators (double buffering), used by two LayerBitmaps.  A
  LayerBaseClient::Surface inheriting BnSurface is created with the two
  bitmaps' heaps, which is quite dumb though.  The two heaps are returned
  to remote on createSurface, which makes graphics op possible on remote
  (possible because of heaps, together with `surface_info_t->bits_offset`)
* surfaceflinger creates a Layer and `addLayer_l`, calls layer's setBuffers,
  initStates, getSurface and getSurfaceData.
* every layerbase is given a id, from sIdentity++
* secure means zero the memory

## Layer

* A layer has `mVertices`.  It is layer's geometry converted to 16.16 fixed point.
  Its values are

    mVertices[0] = (x, y)
    mVertices[1] = (x, y+h)
    mVertices[2] = (x+w, y+h)
    mVertices[3] = (x+w, y)
  where `w` and `h` are positive.  It uses X-window coordinates.
* In `LayerBase::drawWithOpenGL`, `mVertices` is used directly as vertex array.
  Since OpenGL uses cartesian coordinates, the result is the back face with
  upside down image.  With an upside down projection, it gives the wanted result.

SurfaceLinger::threadLoop
* two stage: setClientState changes states; handleTransaction applies states.
  When a global transaction is in action, handleTransaction is not called.
  (Java Surface always does global transaction, see `Surface_openTransaction`)
* handleTransaction: if a connection/surface is made/destroyed, or orientation
  changed, or etc., eTransactionNeeded is set.  if a layer position changed,
  size changed, etc. eTraversalNeeded is set.  All layers marked is
  doTransaction.  surfacelinger then processes itself, and copy mCurrentState to
  mDrawingState.
* handlePageFlip: lockPageFlip all layers, compute mDirtyRegion, unlockPageFlip
  all layers.  flips the front/back buffers.  later after repaint,
  finishPageFlip is called.  By flipping, it changes the front/back indices, as
  can be seen in Layer::post.  When the client locks the surface next time, front
  is memory copied to back.
* what remote draws is wrapped in GGLSurface (LayerBitmap) and turned into
  texture through LayerBase::loadTexture.
* handleRepaint: calls each layer->draw
* hw.flip: eglSwapBuffers

EGLDisplaySurface:
* mapFrameBuffer maps /dev/fb0 and wrap it in GGLSurface.  If fb0 supports
  flipping, mFb[1] points to the other half, otherwise, mFb[1] uses malloc()
* no MemoryDealer -> no one, except itself, directly draws to fb
* mBlitEngine is set if copybit.so is available
* `egl_native_window_t = { .base = intptr_t(mFb[0].data), .offset = intptr_t(mFb[1 - mIndex].data) - egl_native_window_t::base }`
  i.e., native window used for drawing is based by back buffer.
* on swapBuffer, back buffer is copied to front buffer

DisplayHardware:
* DisplayHardwareBase: console management (text/graphics mode, switching to vt7, etc. if not on msm7k?)
			switching VT -> acquire/release screen
* on init, a new EGLDisplaySurface is allocated, and a EGLSurface (window surface) is created.
* on flip, eglSwapBuffer
* flags
  * `SWAP_RECTANGLE_EXTENSION` and `UPDATE_ON_DEMAND` are always set
  * `SLOW_CONFIG` if `EGL_CONFIG_CAVEAT`
  * `BUFFER_PRESERVED` if `EGL_BUFFER_PRESERVED` is set in `EGL_SWAP_BEHAVIOR`
  * `NPOT_EXTENSION` if `GL_ARB_texture_non_power_of_two`
  * `DRAW_TEXTURE_EXTENSION` if `GL_OES_draw_texture`
  * `DIRECT_TEXTURE` if `GL_ANDROID_direct_texture`
* When `relsig` is received in `DisplayHardwareBase`, `screenReleased` is called
  to queue `eConsoleReleased`.  When the event is processed, it gives
  `DisplayHardware` to chance to do anything it wants before ACK.
* Similar to `acqsig`

## `per_client_cblk_t`

* `per_client_cblk_t` has

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
* `(state & eIndex)` gives the buffer the client can use (back buffer).  It is
  complicated by `eFlipRequested`, `eNextFlipPending`, and `eBusy`.
  If some (comibnatin of) the flags are set, the client should wait before it
  can start using.
* `lock_layer` waits until one buffer is available, locks the specified layer by
  setting `eLocked` flag and returns back buffer (0 or 1)
* the mutex is only used for waiting on condition (cv.wait)
* `unlock_layer` unsets `eLocked` flag
* `unlock_layer_and_pos` sets `eFlipRequested` and/or `eNextFlipPending` and
  unsets `eLocked`
* `eLocked` indicates that the client is using it.  It prevents resize to happen
  on the server side, instead, the request is rescheduled by setting
  `eRestartTransaction`
* `eBusy` indicates that the server is flipping (between `lockPageFlip` and
  `finishPageFlip`).

## SurfaceComposerClient

* There are 4 types of CBLK: `surface_flinger_cblk_t`, `display_cblk_t`,
  `per_client_cblk_t`, and `layer_cblk_t`
* `lockSurface` locks and returns the info of the back buffer.  Current dirty
  region is copied from front buffer to back buffer, and is updated to new one.
* `unlockAndPostSurface` sends current dirty region to server (by storing in
  `layer_cblk_t`), calls `unlock_layer_and_post` and signal server.  If there is
  swap rectangle, it is sent instead of dirty region.
* The rest deals with states, which is sent to server through `setState`

## Surface

* For 2D usage, `Surface` wraps the underlying buffer in a `SkBitmap` and
  operates on a `SkCanvas`.  See `Surface_lockCanvas`.
* For 3D usage, `Surface` wraps the underlying buffer in a
  `EGLNativeWindowSurface` and operates in a `GL` context.  The
  `EGLNativeWindowSurface` is passed to `eglCreateWindowSurface` and the
  returned `EGLSurface` is made current in `EGL`.  See `eglCreateWindowSurface`
  in `EGLImpl.java`.

## Dirty Region

* `Layer` has `mPostedDirtyRegion`, as sent by client when `unlockAndPost`
* `LayerBase` has
  * `visibleRegionScreen`, the region which is visible by the user
  * `transparentRegionScreen`, the region hinted by client which should be
    considered transparent, i.e., should be clipped out
  * `coveredRegionScreen`, the region covered by upper layers
* `transparentRegionScreen` is not part of `visibleRegionScreen`
* `SurfaceFlinger`'s regions
  * `mDirtyRegion`: the dirty region decided in `handlePageFlip`.
    It is added the union of all layers' dirty regions
  * `mWormholeRegion`: screen region minus opaque region.  That is, the region
    not covered by any layer.  It is usually a bug if non-empty.  Painted black.
  * `mInvalidRegion`: the region used by DisplayHardware::flip to perform
                  copy-back optimization.  (copy-back: instead of copy back
                  buffer to front before eglSwapBuffer, switch back and front,
                  and copy from back to front?)
* `DisplayHardware` (`EGLDisplaySurface` actually) is double-buffered to avoid
  flicker.  `SurfaceFlinger`'s layers are double-buffered to avoid lock
  contension between server and client.
* computeVisibleRegions: compute and set new visible region, rarely called.
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

* When a client sets the states, the new states are remembered in
  `mCurrentState` of a layer and the surfaceflinger is signaled.
* The main thread calls `handleTransaction` to copy `mCurrentState` to
  `mDrawingState`.
* It then calls `handlePageFlip` to handle front/back buffers swapping.
* Finally, `composeSurfaces` are called to draw.
* There should be hooks for
  * input event
  * state change
  * buffer swapping not needed because it is implied by drawing
  * create/destroy of layers
  * `composeSurfaces`
  * layer drawing
* Conversely, when an effect wants to schedule a work, it calls `signalEvent`
* how to draw a layer?
  * update geometry and draw geometry with window texture
  * `uploadGeometry`
    * index buffer and vertex buffer
  * `drawGeometry`
    * with layer state
    * with user transform and attrib
  * `draw` calls both
* `drawWithOpenGL`
  * framebuffer has no alpha channel; layers use pre-multiplied alpla
  * `needsBlending` returns true if the pixel format has alpha channel
  * 
