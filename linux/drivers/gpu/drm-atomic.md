DRM atomic modesetting
======================

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
