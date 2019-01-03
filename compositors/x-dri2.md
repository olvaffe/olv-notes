DRI2
====

## `ScheduleSwap`

* kernel provides `drmWaitVBlank` to (sync or async) wait for a vblank event.
* kernel provides `drmModePageFlip` to schedule a page flip for the next vblank.
* Together, userspace can implement a asynchronous page flip for any vblank
* Upon `glXSwapBuffers`, the server copies the back buffer to the front one
  on DDX without page flipping.  On newer DDX with page flipping, the server
  calls video driver's `ScheduleSwap`.
* On intel, `ScheduleSwap` may schedule either a swap or a flip.  Either way, it
  will schedule an async wait on a vblank event.  When the vblank event happens,
  `I830DRI2FrameEventHandler` is called.  In the case of flip, it will exchange
  the buffers and do a `drmModePageFlip`;  In the case of swap, it will either
  exchange the buffers or copy the back buffer to the front.
  * By exchanging the buffers, it points the front Pixmap to the bo of the back
    pixmap; and points the back Pixmap to the bo of the front pixmap.  This
    makes sure that the front/back Pixmaps still kepp the drawable IDs, while
    they point to different BOs.

## `ScheduleSwap` and radeon

* (`glXSwapBuffers` flushes and schedules swaps; swap is expected to happen after
   rendering)
* `radeon_dri2_schedule_swap` schedules a `DRI2FrameEventPtr` with type
  `DRI2_FLIP` or `DRI2_SWAP`.  It inserts a vblank wait.
* (the client is notified of invalid buffers; validate will return swapped
   buffers hopefully)
* `radeon_dri2_frame_event_handler` is called after the vblank happened and it
  calls `radeon_dri2_schedule_flip`
  * `drmModeAddFB` is called on the current back
  * `drmModeRmFB` will be called on the current front after flip
  * `drmModePageFlip` is called
* `radeon_dri2_flip_event_handler` is called after the flip happened 
* root pixmap
  * `RADEONScreenInit_KMS`
  * GC op is mapped to exa operations
  * no mapping of bo is required generally; in case a fallback does be needed,
    `RADEONPrepareAccess_CS` is called to map the bo
