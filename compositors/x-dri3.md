# DRI3

 - `DRI3PixmapFromBuffer`: make the prime fd the backing store of a pixmap
 - `DRI3BufferFromPixmap`: get the prime fd of the backing store of a pixmap
 - `DRI3FenceFromFD`: make the xshmfence fd the backing store of a server fence
 - `DRI3FDFromFence`: get the xshmfence fd of the backing store of a server
                      fence

# Present

 - `PresentPixmap`
   - make the pixmap the backing store of the window, to be visible at the
     specified time
   - The server creates a vblank object, to be executed at the specified
     vblank in `present_scmd_pixmap`. At the specified vblank,
     `present_execute` is called.  When the window/pixmap is fullscreen and
     meets the HW requirements, a pageflip is executed.  Otherwise, a copy is
     executed.
 - A pageflip usually involves
   - `DRM_MODE_PAGE_FLIP_EVENT`
   - with atomic support, call `drmModeAtomicCommit` with `DRM_MODE_ATOMIC_NONBLOCK`
   - otherwise, `drmModePageFlip`
 - `ms_present_queue_vblank` request a event to be delivered at the specified
   vblank.  On virtio, where there is no vblank support, the fucntion returns
   false.  The present extension handles the failure by executing the present
   immediately.  Even there is no vblank support, a `DRM_EVENT_FLIP_COMPLETE`
   event is still delievered after the pageflip takes place.

# Atomic Commit

 - it always starts by allocating a `drm_atomic_state`
 - all commit parameters are saved into `drm_atomic_state`
 - a fence and a vblank event are allocated for signaling
 - `atomic_commit` callback is called; there is the generic
   `drm_atomic_helper_commit`
