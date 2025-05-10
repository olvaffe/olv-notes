DRM vblank
==========

## vblank

- HW
  - vblank signal, sent after hw has latched the registers and new values can
    be programmed
  - flip signal, sent after a flip completes, usually some time (hundreds of
    microseconds) after the vblank signal
  - vblank counter, incremented every vsync (even when the interrup is
    disabled)
  - scanout position, which x/y position is being scanned out
- driver
  - `get_vblank_counter` returns the raw hw vblank counter
  - `get_scanout_position` returns the x/y scanout position as well as
    cpu timestamps before and after the reading position
    - x/y are adjusted to be negative if inside vblank
  - `get_vblank_timestamp` returns the timestamp at (0, 0).  If called during
    vblank, the timestamp is in the future.
    - `drm_crtc_vblank_helper_get_vblank_timestamp` implements the callback
      generically
    - it subtracts "(x, y) / crtc clock rate" from the current time
    - because x/y are negative in vblank, the returned time can be in the
      future
  - on vblank signal, `drm_crtc_handle_vblank` or `drm_handle_vblank` must be
    called
  - on flip signal, `drm_crtc_send_vblank_event` should be called if there is
    an event to send
    - or call `drm_crtc_arm_vblank_event` during register update, which will
      send the event on vblank signal
- vblank core
  - `drm_update_vblank_count` uses the driver callbacks to update the cooked
    counter / timestamp
    - this is normally called once per vblank from `drm_crtc_handle_vblank`
  - all the other functions use the cooked values
- event
  - when `DRM_MODE_PAGE_FLIP_EVENT` is set in atomic or flip ioctl, atomic
    core creates a `drm_pending_vblank_event` and the driver takes the event
  - on flip signal, the driver calls `drm_crtc_send_vblank_event`
  - the driver can also make use of `drm_crtc_arm_vblank_event`, to send the
    event automatically on vblank signal

## Events

- Kernel sends three types of events to the userspace
  - `DRM_EVENT_VBLANK`, in response to `DRM_IOCTL_WAIT_VBLANK` with
    `_DRM_VBLANK_EVENT` bit
  - `DRM_EVENT_FLIP_COMPLETE`, in response to `DRM_IOCTL_MODE_PAGE_FLIP` or
    `DRM_IOCTL_MODE_ATOMIC`, with `DRM_MODE_PAGE_FLIP_EVENT` bit
  - `DRM_EVENT_CRTC_SEQUENCE`, in response to `DRM_IOCTL_CRTC_QUEUE_SEQUENCE`
- `drmWaitVBlank`
  - there is a counter for the number of vblanks since the system booted
  - `_DRM_VBLANK_ABSOLUTE` waits until the counter matches the specified number 
  - `_DRM_VBLANK_RELATIVE:` waits for the number of vblanks specified
  - `_DRM_VBLANK_SECONDARY` selects whether crtc 0 or 1 should be used
  - `_DRM_VBLANK_NEXTONMISS` waits for next vblank if the specified one is
    missed
  - Unless `_DRM_VBLANK_EVENT` is set, the caller waits for the specified
    vblank.  When it happens, the current time of day and the current vblank is
    returned.
  - When `_DRM_VBLANK_EVENT` is set, instead of blocking, the call returns
    immediately.
    - a pending event is added to `dev->vblank_event_list`
    - driver calls `drm_handle_vblank` in its IRQ handler to move the event to
      `file_priv->event_list`
    - read() on the fd returns the event
- `drmModePageFlip`
  - see d91d8a3f88059d93e34ac70d059153ec69a9ffc7
  - is handled by driver, such as `intel_crtc_page_flip`
  - the pending event is NULL unless `DRM_MODE_PAGE_FLIP_EVENT` is set
  - the current fb (`work->old_fb_obj`) is remembered
  - the new fb (`work->pending_flip_obj`) is pinned and fenced, and
    `i915_gem_object_flush_write_domain` is called
    - note that a flush in kernel does not mean to submit the instructions to
      GPU.  It adds a `MI_FLUSH` instruction so that the GPU would flush its
      caches.
  - a `MI_DISPLAY_FLIP` instruction is emitted
    - after flush, so that all renderings are guaranteed to be done and hit the
      memory by the time flip happens
  - before the flip is done, `obj->pending_flip` is non-zero.  If the obj is
    used by subsequent rendering before the flip happens,
    `i915_gem_wait_for_pending_flip` is called to wait for the flip.
  - after the flip is done, as notified by IRQ, `intel_finish_page_flip` is
    called.  if a pending event was given, it is added to
    `file_priv->event_list`.  The old fb is unpinned.
- `drmModeSetCrtc`
  - `drm_mode_setcrtc -> drm_crtc_helper_set_config -> intel_pipe_set_base`
  - the new fb is pinned and fenced
  - `i915_gem_object_set_to_display_plane` emits flush and waits, and finally
    move the obj to GTT
  - CRTC is programmed immediately
