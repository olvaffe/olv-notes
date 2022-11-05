DRM Modesetting
===============

## Atomic Modesetting Purpose

- <https://lwn.net/Articles/653071/>
- before atomic,
  - there was `DRM_IOCTL_MODE_SETCRTC` to update primary plane (or set mode)
  - there was `DRM_IOCTL_MODE_CURSOR` to update cursor plane
  - then there was `DRM_IOCTL_MODE_PAGE_FLIP` to update primary plane vsync'ed
  - then there was `DRM_IOCTL_MODE_SETPLANE` to update overlay plane
- after atomic,
  - all primary/cursor/overlay planes are just planes
  - a crtc is a pipeline that connects planes (scanout engines) to connectors
    (screens)
- we want to use hw planes (over GPU) for composition because
  - it saves memory bandwidth: no more composition to a final buffer, yuv can
    be directly scanned out than converting to RGB and scaling
  - hw planes are more energy-efficient to convert and scale
- we need atomic modes setting because
  - planes have complicated restrictions: we want all or nothing
  - old apis are not vsync'ed
- `DRM_IOCTL_MODE_ATOMIC` is used for two different operations
  - atomic mode setting: the screen might go black for seconds
  - atomic plane update: update buffers without changing the mode
    - this is called "nuclear page flip" sometimes
- note that atomic modesetting is disabled for X
  - see how `DRM_CLIENT_CAP_ATOMIC` is handled in `drm_setclientcap`

## KMS Datapaths

- framebuffers feed data into planes, 1:1
- planes feed data into crtc, N:1
- crtc feed data into encoders, 1:N
  - encoders should be an implementation detail, but they are unfortunately a
    part of the public APIs
- encoders feed data into connectors, 1:1
  - there might be a chain of brdiges between the encoder and the connector
- connectors feed data into the physical displays
- in an atomic update, we can change all datapaths, or just a datapth, or just
  a sub-datapath.  E.g., we can update just one `fb->plane->crtc` subpath for
  a pageflip
- to change a datapath end-to-end, it usually requires
  - disable output
    - `->atomic_disable` the bridge chain, starting from the tail (which
      connects to the connector)
    - `->atomic_disable` the encoder
    - `->atomic_post_disable` the bridge chain, starting from the head (which
      connects to the encoder)
    - `->atomic_disable` the crtc
  - change mode
    - `->mode_set_nofb` the crtc
    - `->atomic_mode_set` the encoder
    - `->mode_set` the bridge chain, starting from the head
  - update plane
    - `->atomic_begin` the crtc
    - `->atomic_update` (or `->atomic_disable`) the plane
    - `->atomic_flush` the crtc
  - enable output
    - `->atomic_enable` the crtc
    - `->atomic_pre_enable` the bridge chain, starting from the tail
    - `->atomic_enable` the encoder
    - `->atomic_enable` the bridge chain, starting from the head

## Atomic Modesetting

- `DRM_IOCTL_MODE_ATOMIC`
- we will have many components (crtcs, planes, connectors) to lock down, and
  we don't want a global lock, it necessitates `ww_mutex`
- we colloect all information needed for the atomic update in `drm_atomic_state`
  - for each component included in this atomic update, we lock it down, we
    duplicate its state, and we modify only the duplicated state first
  - a crtc or a connnector might have an out fence.  It is a fence that the
    atomic update must signal after the update completes (usually on next
    vsync)
  - a plane might have an in fence.  It is a fence that the atomic update must
    wait before updating the plane
- with `drm_atomic_state` ready, the driver can look at it and decide whether
  it is doable
- when there are plane out fences or the page flip event is requested
  - we create a `drm_pending_vblank_event` for each crtc interests in it
  - the event will be delivered when the atomic update completes and signals
    the plane out fences
- now is time to call into the driver `->atomic_commit` hook.  If the HW is
  very standard, the driver can use `drm_atomic_helper_commit`
  - `drm_atomic_helper_setup_commit` to set up primitives that help track the
    progress of this commit
    - also check the progress of the previous commit; wait for it if needed
  - `drm_atomic_helper_prepare_planes` to prepare fbs
    - pin the pages of the new fb
    - collect fences for the old (before we can unpin) and new fbs that it
      will wait
  - schedule commit
    - SW should wait for in fences or implicit/internal in fences to signal
      before updating registers
    - modeset registers can be single-buffered or double-buffered.  With
      double-buffered ones, they can be set at any time and will be latched on
      vsyncs; with single-buffered ones, SW should set them on next vsync
    - we don't want atomic update to be blocked so we schedule the commit to
      a workqueue
- finally, return the out fence fds to the userspace
- in the workqueue that does the blocking commit, if the HW is very
  standard, it can
  - `drm_atomic_helper_wait_for_fences` to wait for all fences to signal
  - `drm_atomic_helper_wait_for_dependencies` to wait for preceding commits
  - `drm_atomic_helper_commit_tail`
    - `drm_atomic_helper_commit_modeset_disables`
      - `->atomic_disable` each crtc
    - `drm_atomic_helper_commit_planes` to update the registers
      - `->atomic_begin` each crtc
      - `->atomic_update` each plane
      - `->atomic_flush` each crtc
    - `drm_atomic_helper_commit_modeset_enables`
      - `->atomic_enable` each crtc
    - `drm_atomic_helper_commit_hw_done` to update the progress
    - `drm_atomic_helper_wait_for_vblanks` to wait until next vsync
    - `drm_atomic_helper_cleanup_planes` to clean up after the commit
  - `drm_atomic_helper_commit_cleanup_done` to update pregress
  - no failure allowed
- In userspace API
  - `DRM_MODE_ATOMIC_NONBLOCK` means `DRM_IOCTL_MODE_ATOMIC` should be
    non-blocking
  - `DRM_MODE_PAGE_FLIP_ASYNC` means tearing is fine; the atomic commit does
    not need to happen on vsyncs

## Atomic Modesetting as the Driver Mechanism

- `DRM_IOCTL_MODE_PAGE_FLIP` can be implemented with the generic
  `drm_atomic_helper_page_flip`
  - it does a non-blocking atomic commit to udpate a `fb->plane->crtc`
    sub-datapth
- `DRM_IOCTL_MODE_DIRTYFB` can be implemented with the genric
  `drm_atomic_helper_dirtyfb`
  - it does a blocking atomic commit to udpate the `FB_DAMAGE_CLIPS` property
    of a plane

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

## dumb

- `DRM_IOCTL_MODE_CREATE_DUMB`
  - in: width, height, bpp
  - out: handle, pitch, size
- `DRM_IOCTL_MODE_ADDFB`
  - in: width, height, bpp, pitch, depth
- `DRM_IOCTL_MODE_MAP_DUMB` return an fake offset for mmap
  - for software rendering
- `DRM_CAP_DUMB_BUFFER` checks if the driver supports dumb
- `DRM_CAP_DUMB_PREFERRED_DEPTH` returns the preferred depth
  - static, usually 24 (because alpha is ignored)
  - 0 on virtio-gpu
- `DRM_CAP_DUMB_PREFER_SHADOW` returns whether shadow is preferred
  - static, usually true (because dumb might be in vram and/or uncached)
  - false on virtio-gpu
- xorg-modesetting without gbm does
  - when preferred depth is 8 or 16, depth=16, bpp=16, kbpp=0
  - otherwise,
    - depth=24, bpp=32, kbpp=0 if bpp=32 dumb and fb is supported
    - depth=24, bpp=32, kbpp=24 and forces bpp=32 shadow
- i915 assumes the dumb bo format to be `DRM_FORMAT_C8`, `DRM_FORMAT_RGB565`,
  or `DRM_FORMAT_XRGB8888` depending on bpp

## KMS object properties

- `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` can get all properties of a KMS object
  - in: object id and object type `DRM_MODE_OBJECT_*`
  - out: an array of (32-bit prop id, 64-bit prop val) pairs
  - each prop id seems to be globally unique (see below)
- `DRM_IOCTL_MODE_GETPROPERTY` can get info about a prop id
  - in: prop id
  - out: name and values
    - name is well-defined and is a part of the API
    - values depends on the flags
      - for `DRM_MODE_PROP_RANGE`, values are 64-bit pairs
      - for `DRM_MODE_PROP_ENUM`, values are an array of
        (well-defined enum name, 64-bit enum id)
      - for `DRM_MODE_PROP_BITMASK` is similar to `DRM_MODE_PROP_ENUM`
      - for `DRMO_MODE_PROP_BLOB`, values are an array of 32-bit blob ids that
      	can be passed to `DRM_IOCTL_MODE_GETPROPBLOB`
  - remember that these are property infos; the current value is returned by
    `DRM_IOCTL_MODE_OBJ_GETPROPERTIES`
- for example, a `DRM_MODE_OBJECT_PLANE` object has a property `type`
  indicating whether the plane is primary, cursor, or overlay
  - we list all properties first to get prop ids and prop values first
  - for each prop id, we get the info.  We are interested in the prop id whose
    name is `type`
  - the info should also contains a list of (enum name, enum val) enums
  - by comparing the prop value with the enum vals, we can find out the
    current enum name, which should be one of `Primary`, `Overlay`, or
    `Cursor`

## DRM modifiers

- To kernel,
  - `DRM_FORMAT_MOD_INVALID` (0x00ffffffffffffff) is truely invalid
  - `DRM_FORMAT_MOD_LINEAR` (0x0) is truely linear
  - in addfb2, when `DRM_MODE_FB_MODIFIERS` is not set, modifiers are not
    (explicitly) given.  modifiers array are required to be zeroed, but it is
    not the same as `DRM_FORMAT_MOD_LINEAR`
    - `intel_framebuffer_init` changes modifiers[0] when not explicitly given
      before calling `drm_helper_mode_fill_fb_struct` to set `fb->modifier`
  - a plane's `IN_FORMATS` never contains `DRM_FORMAT_MOD_INVALID`
- To EGL,
  - in eglCreateImage, when modifier is not explicitly given, it is considered
    to be `DRM_FORMAT_MOD_INVALID` and means impl-defined (similiar to addfb2
    without `DRM_MODE_FB_MODIFIERS`)
  - however, when `DRM_FORMAT_MOD_INVALID` is explicitly given, it becomes
    invalid
- xorg-modesetting allocates front buffers that are rendered to by glamor and
  scanned out by display engine
  - traditionally, set `GBM_BO_USE_RENDERING | GBM_BO_USE_SCANOUT` and be done
  - when supported, use modifiers instead
  - during crtc init, xorg-modesetting finds the best plane for the crtc
    - `drmmode_crtc_create_planes` also decodes the `IN_FORMATS` prop of the
      plane to get the list of supported formats and modifiers
    - virtio-gpu supports two planes
      - primary supports XR24
      - cursor supports AR24
    - i915 supports 9 planes
      - three primary supports CB RG16 XR24 XB24 XR30 XB30 XB4H (LINEAR or x-tiled)
      - three overlay supports XR24 XB24 XR30 XB30 XR4H XB4H YUYV YVYU UYVY VYUY (LINEAR or x-tiled)
      - three cursor supports AR24 (LINEAR)
  - bo allocation will pass in the modifiers supported by the crtc/plane
- glamor needs to allocate a pixmap or dmabuf-to-pixmap or pixmap-to-dmabuf
  - for alloc, `glTexImage2D`
  - for import, `gbm_bo_import`, `eglCreateImageKHR`, and
    `glEGLImageTargetTexture2DOES`
  - for export, `gbm_bo_create*`, import as pixmap, CopyArea, and replace the
    texture-based pixmap by the bo-based pixmap
  - EGL dmabuf extensions allow glamor to know which modifiers are supported
- gbm supports usage or modifiers, but not both
  - for usage, the dri backend only cares about `GBM_BO_USE_SCANOUT`,
    `GBM_BO_USE_CURSOR`, and `GBM_BO_USE_LINEAR`
  - i965's `intel_create_image_common` uses usage to pick between linear and
    x-tiled; for modifiers, it becomes explicit and can fail (when the
    modifier is not supported)
  - gallium drivers must implement `resource_create_with_modifiers` to support
    modifiers
  - gallium's `dri2_create_image_common` always sets `PIPE_BIND_RENDER_TARGET`
    and `PIPE_BIND_SAMPLER_VIEW`; it uses usage to set scanout/linear/cursor
    binds, which are not set with modifiers
- EGL supports modifiers
  - when importing a dma-buf, no modifier means implementation-defined
    modifier; `DRM_FORMAT_MOD_INVALID` is not a valid explicit modifier

## DRM formats

- `DRM_IOCTL_MODE_ADDFB2` takes a format instead of bpp/depth.  It also
  supports modifiers and planar formats.
  - internally, `drm_mode_legacy_fb_format` translates bpp/depth to format and
    `DRM_IOCTL_MODE_ADDFB` can take the same path as addfb2 does
- modifiers must be explicitly enabled by setting `DRM_MODE_FB_MODIFIERS` flag

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
