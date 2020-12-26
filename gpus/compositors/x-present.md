X Present Extension
===================

## Overview

- modesetting driver calls `present_screen_init` to enable screen-flip present
  support
  - only full-screen windows can be flipped
  - the rests are copied
- xwayland calls `present_wnmd_screen_init` to enable window-flip present
  support
  - all windows flip

## GLX

- Present was designed to support `GLX_OML_swap_method` and
  `GLX_OML_sync_control`
- `GLX_OML_swap_method`
  - a swap can be `GLX_SWAP_EXCHANGE_OML` or `GLX_SWAP_COPY_OML`
- `GLX_OML_sync_control`
  - UST, Unadjusted System Time
    - system-defined monotonic timestamp of the last vblank
  - MSC, Media Stream Counter
    - incremented every vblank
  - SBC, Swap Buffer Counter
    - associated with the window and
    - incremented at the completion of each buffer swap (e.g., copy to
      frontbuffer completed or hardware register that swaps written) 
  - `glXGetSyncValuesOML` queries the current values of all three counters
  - `glXGetMscRateOML` returns vblank frequency in hertz
  - `glXSwapBuffersMscOML` performs a buffer swap at the target MSC
    - no implied `glFlush`
    - however, buffer swap won't start until prior rendering commands have
      completed
    - when prior rendering commands have completed, it compares the current
      MSC and with the target MSC
      - if current < target, it waits until target to swap
      - if current >= target, it waits until current+1 to swap
  - `glXWaitForMscOML` blocks the caller until the target MSC
  - `glXWaitForSbcOML` blocks the caller until the target SBC
- on modern Linux, `drmCrtcGetSequence` returns sequence (vblank counter, for
  MSC) and timestamp (for UST)

## Life of a PresentPixmap request on Xorg

- when X receives `PresentPixmap`, it dispatches to `proc_present_pixmap`
- `proc_present_pixmap` calls `present_pixmap` which calls
  - `present_scmd_pixmap`, if Xorg
  - `present_wnmd_pixmap`, if Xwayland
- `present_scmd_pixmap`
  - it calls `present_get_ust_msc` to get the current msc/ust
    - this calls `ms_present_get_ust_msc` and `drmCrtcGetSequence`
  - it calls `present_vblank_create` to createa a `present_vblank_ptr` to
    describe the request
  - it adds the `present_vblank_ptr` to `present_exec_queue`
  - it then `present_queue_vblank` or `present_execute` depending on whether
    the operation happens in a future msc or now
    - if in the future, `present_queue_vblank` calls `ms_present_queue_vblank`
      and `drmCrtcQueueSequence` to get a notification for the target msc.
      When notified, it calls `ms_present_vblank_handler` which calls
      `present_event_notify` which calls `present_execute`
- `present_execute` executes the request (`present_vblank_ptr`)
  - if flipping, the request is moved to `present_flip_queue` list.
    `present_flip` calls `ms_present_flip` to schedule the flip.  Note that
    `exec_msc` is `target_msc` minus 1 in case of flipping.  When the flip
    takes place at `target_msc`, `ms_present_flip_handler` is called to call
    `present_event_notify`
  - if copying, `present_execute_copy` calls `present_copy_region` and
    `ms_present_flush` to ask GPU to copy.  Because the pixmap is no longer
    needed by the server, `present_pixmap_idle` is called to mark the fence
    triggered and to send `PresentIdleNotify`
  - there are many reasons that the request needs to be requeued for later
    - in case of flipping, another flip has already been scheduled
    - the wait fence associated with the pixmap hasn't been triggered
    - errors and falling back to copying (because flipping happens at target
      msc minus 1 and rescheduling is needed)
- `present_execute_post` or `present_flip_idle` completes the request
  - for copying, `present_execute_post` sends a `PresentCompleteNotify` to the
    client and destroys the request
  - for flipping, `present_flip_notify` calls `present_pixmap_idle` on the
    previous pixmap to mark the fence triggered and send `PresentIdleNotify`.
    It also calls `present_vblank_notify` to send `PresentCompleteNotify`.
    The request is then destroyed
    - note that this marks the PREVIOUS pixmap idle
