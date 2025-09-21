DRM i915
========

## Initialization

- `i915_pci_probe` is the entrypoint
  - `intel_info` is from the pci id table `pciidlist`
    - for mtl, it's `mtl_info`
  - `i915_driver_probe` calls various init functions
- `i915_driver_create` creates `drm_i915_private`, a subclass of
  `drm_device`
  - the `drm_driver` is `i915_drm_driver`
  - `intel_display_device_probe` probes the display ver and feats
- `i915_driver_early_probe` initializes states that do not require hw
  access
  - detects ip versions/steppings
  - initializes locks, wqs, ttm, root gt, gem, etc.
- `intel_vgpu_detect` detects GVT-g
- `intel_gt_probe_all` initializes gt
- `i915_driver_mmio_probe` initializes gmch, display, and gt mmio
- `i915_driver_hw_probe` initializes states that require hw access
  - `intel_gt_assign_ggtt` creates/assigns `i915_ggtt` for each gt
    - global graphics translation table for `va->pa` translations
- `intel_display_driver_probe_noirq`
  - `drm_vblank_init` initializes vblank
  - `intel_bios_init` initializes bios
  - `intel_power_domains_init` initializes display power domains
  - `intel_dmc_init` initializes dmc and loads `i915/mtl_dmc.bin`
  - `intel_mode_config_init` initializes `drm_mode_config`
- `intel_irq_install` requests irq
- `intel_display_driver_probe_nogem` does more display init
  - `intel_wm_init` initializes watermark
  - `intel_crtc_init` initializes crtcs
- `i915_gem_init` initializes gem
  - `intel_uc_fetch_firmwares` calls `__uc_fetch_firmwares` to load
    `i915/mtl_guc_70.bin`, `i915/mtl_huc_gsc.bin`, and
    `i915/mtl_gsc_1.bin`
  - `intel_gt_init` initializes gt
  - `intel_engines_driver_register` registers engines
- `intel_pxp_init` initializes pxp (for protected content)
- `intel_display_driver_probe`
  - `intel_initial_commit` performs the first atomic commit
  - `intel_fbdev_init` initializes `intel_fbdev` for fbdev emu
- `i915_driver_register` registers i915 to various subsystems
  - `i915_gem_driver_register` registers mm shrinker
  - `i915_pmu_register` registers pmu (performance monitoring unit)
  - `drm_dev_register` registers the drm device/driver
  - `i915_debugfs_register` populates debugfs
  - `i915_setup_sysfs` populates sysfs
  - `intel_gt_driver_register` adds gt as `auxiliary_device`
  - `intel_pxp_debugfs_register` populates debugfs for pxp
  - `i915_hwmon_register` registers a hwmon for dgpu
  - `intel_display_driver_register` registers display

## `intel_device_info` and `intel_runtime_info`

- `INTEL_INFO(i915)` returns the per-device `intel_device_info`
  - it is defined in the pci id table, `pciidlist`, and is initialized by
    `intel_device_info_driver_create` during `i915_driver_create`
  - it is read-only
- `INTEL_INFO(i915)` returns the per-device `intel_runtime_info`
  - it is initialized from `intel_device_info` in
    `intel_device_info_driver_create`
  - it is read-write and is modified by other init functions
- `intel_device_info` for mtl is `mtl_info`
  - `.__runtime.graphics.ip.ver` is 12
  - `.__runtime.graphics.ip.rel` is 70
  - `.__runtime.media.ip.ver` is 13
  - `.has_gmd_id` is 1
  - `.has_llc` is 0
- `intel_runtime_info` for mtl
  - `intel_ipver_early_init` initializes `runtime->graphics.ip` and
    `runtime->media.ip`
    - mtl reads `GMD_ID_GRAPHICS` and `GMD_ID_MEDIA`
  - `intel_step_init` initializes `RUNTIME_INFO(i915)->step`
    - `STEP_XY` where `X` is `[A, J]` and `Y` is `[0, 3]`

## `intel_display_device_info` and `intel_display_runtime_info`

- `DISPLAY_INFO(i915)` returns the per-device `intel_display_device_info`
  - it is defined in `intel_display_ids` and `gmdid_display_map`, and is
    initialized by `intel_display_device_probe` during `i915_driver_create`
  - it is read-only
- `DISPLAY_RUNTIME_INFO(i915)` returns the per-device `intel_display_runtime_info`
  - it is initialized from `intel_display_device_info` in
    `intel_display_device_probe`
  - it is read-write and it modified by other init functions
- `intel_display_device_info` for mtl is probably `xe2_lpd_display`
- `intel_display_runtime_info` for mtl
  - `intel_display_device_probe` initializes `DISPLAY_RUNTIME_INFO(i915)->ip`
    from `GMD_ID_DISPLAY`

## Display Planes

- `intel_crtc_init` calls `skl_universal_plane_create` or
  `intel_cursor_plane_create` to create planes
  - `drm_crtc_init_with_planes` initializes the `drm_crtc` with planes
- `skl_universal_plane_create`
  - it initializes plane caps and callbacks
  - `icl_get_plane_formats` gets supported formats
  - `intel_fb_plane_get_modifiers` gets supported modifiers
  - `drm_universal_plane_init` initializes the `drm_plane`
- gem object pinning
  - `i915_gem_object_pin_to_display_plane` pins the gem object, with this call
    stack
    - `intel_atomic_commit`
    - `intel_atomic_prepare_commit`
    - `intel_prepare_plane_fb`
    - `intel_plane_pin_fb`
  - `i915_gem_object_set_cache_level` sets the cache level to
    `I915_CACHE_NONE`
    - this calls `i915_gem_object_unbind` to unbind the bo from all gpu vmas

## sysfs

- `/sys/class/drm/card0`
  - `gt_act_freq_mhz` is the current actual freq
  - `gt_cur_freq_mhz` is the current selected freq
  - `gt_RP0_freq_mhz` is the max non-boost freq (render p-state 0)
  - `gt_RP1_freq_mhz` is the most efficient freq (render p-state 1)
  - `gt_RPn_freq_mhz` is the lowest possible freq (render p-state n)
  - writables
    - `gt_boost_freq_mhz` is the requested boost freq
    - `gt_max_freq_mhz` is the requested max freq
    - `gt_min_freq_mhz` is the requested min freq
  - `rps_work` handles freq change
    - if there is any waiter, it picks `boost_freq`
    - if `GEN6_PM_RP_UP_THRESHOLD`, it increases the freq until
      `max_freq_softlimit`
    - if `GEN6_PM_RP_DOWN_THRESHOLD`, it decreases the freq until
      `min_freq_softlimit`
    - if `GEN6_PM_RP_DOWN_TIMEOUT`, it picks `efficient_freq` and then
      `min_freq_softlimit`
- `/sys/kernel/debug/dri/0`
  - `i915_frequency_info`
    - `Up threshold` is 95% and `Down threshold` is 85%

## i915 modeset

- `intel_modeset_init`
- Depending on the chipset, 1 or 2 pipes (`intel_crtc`) are created.
- Depending on the chipset again, some of `intel_crt_init`, `intel_lvds_init`,
  `intel_sdvo_init`, `intel_hdmi_init`, `intel_dvo_init`, `intel_tv_init` are
  called.  Each function creates a `intel_output`, which consists of
  - a `drm_connector` of types like `DRM_MODE_CONNECTOR_VGA`
  - a `drm_encoder` of types like `DRM_MODE_ENCODER_DAC`
  - `drm_mode_connector_attach_encoder` is called to setthe connector's `encoder_ids`
- each `intel_output`'s encoder is set hardcoded values, depending on output
  type
  - `possible_crtcs`: only the first two bits are set (which pipes supported?)
  - `possible_clones`: bitmaps of supported connectors (connectors are ordered?)
- LVDS is always the second pipe?
- `drmModeSetCrtc` corresponds to `drm_crtc_helper_set_config` in kernel.  It
  finds a best encoder for each given connector, matches given connectors'
  best encoders with the given crtc (and disconnects those not given).
  `drm_crtc_helper_set_mode` and `drm_helper_disable_unused_functions` is
  called.

## dma-buf

- `i915_gem_prime_import` imports a dma-buf
  - `dma_buf_attach`
  - `i915_gem_object_init` inits the obj with
    - `i915_gem_object_dmabuf_ops`
    - `I915_BO_ALLOC_USER`
- `i915_gem_mmap_offset_ioctl` fails to return a fake mmap offset
  - `__assign_mmap_offset` returns `-ENODEV` because
    `i915_gem_object_has_struct_page` returns false
- when `i915_gem_object_pin_pages` pins pages (for hw access), 
  - `i915_gem_object_get_pages_dmabuf` calls `dma_buf_map_attachment` to get
    sgt

## `i915_gem_execbuffer2_ioctl`

- `drm_i915_gem_execbuffer2`
  - `buffers_ptr` and `buffer_count` are an array of `drm_i915_gem_exec_object2`
  - `batch_start_offset` and `batch_len` are the starting offset and size of
    the commands (see `I915_EXEC_BATCH_FIRST`)
  - `DR1` and `DR4` are unused
  - `num_cliprects` and `cliprects_ptr` depend on the flags
  - modern `flags`
    - bit 0..5 selects the queue
      - `I915_EXEC_DEFAULT`, `I915_EXEC_RENDER`, etc.
    - `I915_EXEC_NO_RELOC` is always set (bos are soft-pinned)
    - `I915_EXEC_HANDLE_LUT` is always set (?)
    - bit 13..14 selects the bsd (video) ring
    - `I915_EXEC_BATCH_FIRST` means the commands is in the first bo rather
      than the last
    - `I915_EXEC_FENCE_ARRAY` means `num_cliprects` and `cliprects_ptr` are an
      array of `i915_gem_exec_fence` (binary syncobjs)
    - `I915_EXEC_USE_EXTENSIONS` means `cliprects_ptr` is a list of
      `i915_user_extension`
      - the only extension is `drm_i915_gem_execbuffer_ext_timeline_fences`
        for timeline syncobjs
  - unused `flags`
    - bit 6..7 selects the const addressing mode
      - always `I915_EXEC_CONSTANTS_REL_GENERAL`
    - `I915_EXEC_GEN7_SOL_RESET` is unused (a gen7-specific wa?)
    - `I915_EXEC_SECURE` is unused (allows dangerous cmds)
    - `I915_EXEC_IS_PINNED` is unused
    - `I915_EXEC_RESOURCE_STREAMER` is unused
    - `I915_EXEC_FENCE_IN`, `I915_EXEC_FENCE_SUBMIT`, and
      `I915_EXEC_FENCE_OUT` mean `rsvd2` holds the in/out sync fds
      - they have been obsoleted by syncobjs
  - `rsvd1` is the context id
  - `rsvd2` is the in/out sync fds
- `drm_i915_gem_exec_object2`
  - `handle` is the gem handle
  - `relocs_ptr`, `relocation_count`, and `alignment` are unused (because of
    soft-pin)
  - `offset`
  - modern `flags`
    - `EXEC_OBJECT_WRITE` means the object is written to and needs an
      exclusive implicit fence
    - `EXEC_OBJECT_SUPPORTS_48B_ADDRESS` means the object uses the full 48-bit
      addr (as opposed to be at the first 32-bit addr)
    - `EXEC_OBJECT_PINNED` is always set (soft-pin?)
    - `EXEC_OBJECT_ASYNC` disables implicit fencing
    - `EXEC_OBJECT_CAPTURE` means the contents are saved on hangs (for when
      there is no privacy concern)
  - unused `flags`
    - `EXEC_OBJECT_NEEDS_FENCE` is unused (this is the de-tiling "fence")
    - `EXEC_OBJECT_NEEDS_GTT` is unused
    - `EXEC_OBJECT_PAD_TO_SIZE` is unused
- fences
  - `I915_EXEC_FENCE_ARRAY` provides an array of binary syncobjs
    - `add_fence_array` adds them to `eb->fences`
  - `drm_i915_gem_execbuffer_ext_timeline_fences` provides an array of
    timeline syncobjs
    - `parse_timeline_fences` parses them to `eb->fences`
  - legacy `I915_EXEC_FENCE_IN` or `I915_EXEC_FENCE_SUBMIT` provides a sync fd
  - legacy `I915_EXEC_FENCE_OUT` returns a sync fd
  - `eb_requests_create` creates `eb->requests`
    - each request has a new `rq->fence`
  - `eb_fences_add` adds all fences to the first request
    - `I915_EXEC_FENCE_IN` is awaited by `i915_request_await_dma_fence`
    - `I915_EXEC_FENCE_SUBMIT` is awaited by `i915_request_await_execution`
    - binary and timeline syncobjs are awaited by `await_fence_array`, which
      calls `i915_request_await_dma_fence`
  - `i915_request_await_dma_fence`
    - if the fence is on the same timeline as the requst, it can be skipped
    - if the fence is from a different driver, `i915_request_await_external`
    - if the fence is from i915, it is associated with another request and
      `i915_request_await_request` awaits for it
- implicit fencing
  - unless `EXEC_OBJECT_ASYNC` is set to disable implicit fencing,
    `eb_move_to_gpu` calls `i915_request_await_object` to async-wait the
    implicit fences associated with the objects
  - each request is associated with a new fence (to be signaled on request
    retire), and `_i915_vma_move_to_active` adds the new fence to the objects
    as the implicit fence
  - IOW, implicit fences are always set up by the producer, and they are
    waited by the consumers unless `EXEC_OBJECT_ASYNC` is set
    - they are always set up for kernel memory management (e.g., which objects
      can be swapped out on memory pressure)

## PREAD/PWRITE

- pread/pwrite are reading/write from _CPU_ perspective, according to
  `i915_gem_pwrite_ioctl`.
- `i915_gem_object_get_pages` gets and stores the pages of the backing shmem in
  `struct drm_i915_gem_object`'s `pages`.  `i915_gem_object_do_bit_17_swizzle`
  is called if the object is tiled.
- `i915_gem_object_bind_to_gtt` finds an unused region from
  `dev_priv->mm.gtt_space` and it becomes the `gtt_space` of the object.  It
  thens binds object's pages to its `gtt_space` using `drm_agp_bind_pages`.
- When `i915_gem_mmap_gtt_ioctl` is called, the object's `mmap_offset` is
  determined, and it is bound to gtt.
- Now the user can gtt mmap the object using its `mmap_offset`.  The fault
  handler is `i915_gem_fault`.  It fauls in the pages based on `dev->agp`, with
  the fence reg properly set (when tiled).
- When an object is bound to gtt, it is put in the `inactive_list`.
  - It means it is inactive to the GPU, and can be evicted.
- When an object is pinned, it is removed from the `inactive_list`.
  - It is so that the object is never considered for eviction.
- When evicting something, it looks for inactive object first.  It calls
  `i915_gem_object_unbind` on the first object it finds.
  - If the object is pinned, unbind fails.
  - `agp_unbind_memory` is called to make room in gatt
  - `i915_gem_object_put_pages` is called to return the pages storing the
    contents of the shmem.
  - `drm_mm_put_block` is called to give the region back to `gtt_space`
  - the object is removed from any list

## Tiling

- Tiled address of `Vol_1b_G45_core.pdf`.
- Channel XOR
  - For CPU, bit6 affects none, bit 11 or bit 17
  - For GPU, bit6 affects 
    - none, if no tiling
    - bit 9, if y tiling
    - bit 9 and bit 10, if x tiling
  - When we say bitX affects bitY, it means the bitY of an physical address is
    XORed with bitX of the same address.

## MMAP

- `struct drm_gem_object` is backed by shmem created by `shmem_file_setup`.
  - It has `shmem_file_operations` as its `f_op`.
- `i915_gem_mmap_ioctl` calls `do_mmap` on the underlying shmem.
  - a `struct vm_area_struct` is created
  - `file->f_op->mmap` is called on the vma.  It sets `vma->vm_ops` to
    `shmem_vm_ops`.
  - Whenever the user acceses the memory, the pages are faulted in through
    `shmem_fault`.
- `i915_gem_mmap_gtt_ioctl` binds the backing pages to agp through
  `drm_agp_bind_pages`.
  - When the user mmap it, an vma with `i915_gem_vm_ops` is created.
  - the fault function sets up the pte to access the faulted in page from agp.

## `agp_bind_memory`

- It takes a `struct agp_memory` and a `off_t`.
  - The former gives the gart addresses of the pages to be bound
  - The latter gives the offset into the AGP aperture where the map should base
- It calls `bridge->driver->insert_memory`.
- In case of `agp_generic_insert_memory`,
  - It updates the gatt table.
  - gatt table is an array of physical addresses, aligned on page boundaries
  - when accessing through agp aperture, if the address falls in gatt entry i,
    it acceses the memory given by the physical address of entry i.
- E.g. `i915_gem_fault`
  - It knows `page_offset`, the page in the object the user wants to access
  - It knows `gtt_offset`, the first entry of the object in the gatt table
  - It knows `agp->base`, the base of the agp aperture
  - The logical address the user should access is
    `agp->base + gtt_offset + (page_offset << PAGE_SHIFT)`
  - The physical adress the user will access is decided by gatt entry
    `(gtt_offset >> PAGE_SHIFT) + page_offset`.

## Object Status

- Object has 3 status, inactive, active, and flushing
- objects are moved to active list when they are used by a request
- active objects are moved to other lists when the request is retired
  - objects being read are moved to inactive list
  - objects being written are moved to flushing list
- flushing objects will become active before it becomes inactive
  - when a request is added, the domains its commands flush are provided.
    They are used to move flushing objects to active objects, and those objects
    will be moved to inactive list when the request is retired.
- Objects on inactive list might be evicted at any time
  - Pinning an object removes it from the inactive list
  - Unpinning an object adds it to the inactive list if it is inactive

## domains

- a domain indicates two things
  - an object is readable/writable from that domain.
  - an object is assumed to have read/written from/to that domain.
  - the latter decides whether flushing is needed in some places.
- In `i915_gem_execbuffer`, relocs are applied such that objects have pending
  domain changes in `pending_read_domains` and `pending_write_domain`.
  - `i915_gem_object_set_to_gpu_domain`, which is only called in that function,
    reuses those fields too.
  - `i915_gem_flush` is called to add commands to accomplish the pending
    changes.  `i915_add_request` is also called to move flushing objects to
    active list.
  - pending domains are copied to current domains.
- `i915_gem_flush` puts command to the ring buffer to invalidate/flush gpu
  domains
  - A domain might read its cached data.  If we are going to read from that
    domain and we know that some other domains have modified the data, the
    domain should be invalidated.
  - A domain might write to its cache.  If we are or know other domains are
    going to read the data, the domain should be flushed first.
- A flushed write domain will not be a write domain
  - it flushes the written data
  - _AND_ it unsets the flag in `write_domain`.
  - this is because a domain is flushed to be read by others.  We don't want it
    to be modified in-between.
- Setting to a domain involves
  - make sure there is no `write_domain` by flushing current write domains
  - make sure the new domain is readable/writable
  - update the domain flags
- `i915_gem_idle` waits the hw to be idle
  - it is called when switching away from current vt or suspending

## execbuffer

- The struct passed to `i915_gem_execbuffer` is basically an array of
  `struct drm_i915_gem_exec_object`.
  - It gives the object in question and its associated relocs
  - Usually, all but the last exec objects have no relocs
  - The last exec object describes the batchbuffer.  Its relocs' targets are
    given in that same array.
- First, the exec object array is copied into the kernel space
  - `exec_list` is the array
  - `relocs` is the relocs collected from each exec object
  - `object_list` is the gem objects corresponding to the exec objects
- The DRM device is locked at this point.
- Each exec object is pinned and relocated by `i915_gem_object_pin_and_relocate`
  - It pins the given object.
  - The object's buffer is patched by the relocs.
    - reloc's offset gives where in the buffer needs to be patched
    - reloc's delta is added by target's `gtt_offset` to form the final value
    - reloc's domains is the pending domains of the target
  - Usually, only the last exec object has relocs.
- Each object is `i915_gem_object_set_to_gpu_domain`.
- It puts commands to move objects to desired domains.
- It asks the GPU to execute by calling `i915_dispatch_gem_execbuffer`.
- `i915_retire_commands` is called to insert flush command to the end
- `i915_add_request` is called
  - to add commands to write a monotonically increasing seqno into
    `I915_GEM_HWS_INDEX` and to interrupt when executed.
  - to record the seqno and add it to `request_list`.
  - to move objects on the flushing list to active list, if they are known to be
    flushed by the commands of this request.
  - to schedule `i915_gem_retire_work_handler` to timeout request.
- All objects are moved to `active_list`.
  - with a seqno denoting which request they are active for.
- All objects are unpinned.
- ...
- The DRM device is unlocked.
- Some of the per-device or per-object variables are used only within the lock.
- later when `i915_gem_retire_requests` is called (1HZ after the request is
  added), objects are moved to inactive list or flushing list.
  - it calls `i915_gem_retire_request` to require requests
  - objects that are modified (has write domain) are moved to flushing list
  - the others are moved to inactive

## GPU hang

- <https://gitlab.freedesktop.org/drm/intel/-/wikis/How-to-file-i915-bugs>
- `CONFIG_DRM_I915_CAPTURE_ERROR`
  - `intel_gt_handle_error` is called on gpu hang
  - `i915_capture_error_state` captures a coredump if this is the first hang
    - it also prints `GPU HANG: ecode %d:%x:%08x` to dmesg
      - `%d` is `GRAPHICS_VER`, aka gen
      - `%x` is a bitmask of all hung engines
- `/sys/class/drm/card0/error`
  - `i915_first_error_state` returns the coredump
  - `i915_gpu_coredump_copy_to_buffer` copies the coredump to userspace buffer
  - `/sys/kernel/debug/dri/0/i915_error_state` is the older name and works
    similarly
  - `aubinator_error_decode` from mesa can be used to decode the dump
    - `intel_error_decode` from igt works too, but has less info

## i915 directory structure

- `display/` is for modesetting
- `gem/` is for gem ioctls
- `gt/` is code that talks to the hw (GT, Graphics Technology)
  - `shaders/` is shader code
  - `uc/` is microcontrollers inside GT
    - there are GuC (graphics microcontroller?) and HuC (HEVC
      microcontroller?)
    - there is also GSC, graphics system controller, which is a chassis
      controller for discrete cards
- `gvt/` is for GVT-g, mediated passthrough based on vfio
- `pxp/` is protected Xe path for gen12+, to establish hardware protected
  session and to manage software session
- `soc/` is for things not really about the gpu
  - `intel_detect_pch` detects the PCH
  - `intel_dram_detect` detects dram types and channels
  - `intel_dram_edram_detect` detects the size of eDRAM
    - embedded dram for gpu, from haswell to coffee lake
  - GMCH: Graphics and Memory Controller Hub
- <https://igor-blue.github.io/2021/02/10/graphics-part1.html>

## Firmwares

- <https://01.org/group/43/downloads/firmware>
  - GuC is designed to perform graphics workload scheduling on the various
    graphics parallel engines. In this scheduling model, host software submits
    work through one of the 256 graphics doorbells and this invokes the
    scheduling operation on the appropriate graphics engine. Scheduling
    operations include determining which workload to run next, submitting a
    workload to a command streamer, pre-empting existing workloads running on
    an engine, monitoring progress and notifying host software when work is
    done.
  - DMC provides additional graphics low-power idle states. It provides
    capability to save and restore display registers across these low-power
    states independently from the OS/Kernel.
  - HuC is designed to offload some of the media functions from the CPU to
    GPU.  These include but are not limited to bitrate control, header
    parsing. For example in the case of bitrate control, driver invokes HuC in
    the beginning of each frame encoding pass, encode bitrate is adjusted by
    the calculation done by HuC. Both the HuC hardware and the encode hardcode
    reside in GPU.  Using HuC will save unnecessary CPU-GPU synchronization.
- `intel_uc_fw_init_early` calls `__uc_fw_auto_select` to select the fw
  - `INTEL_GUC_FIRMWARE_DEFS` and `INTEL_HUC_FIRMWARE_DEFS`
  - take `fw_def(TIGERLAKE,    0, guc_mmp(tgl,  70, 1, 1))` for example
    - `MAKE_GUC_FW_PATH_MMP` expand it to `i915/tgl_guc_70.1.1.bin`
    - mmp stands for major, minor, and patch
  - in another example, `fw_def(ALDERLAKE_P,  0, guc_maj(adlp, 70, 5))`,
    - `MAKE_GUC_FW_PATH_MAJOR` expands it to `i915/adlp_guc_70.bin`
    - that is, the firmware promises backward compat within the same major
      version
  - actually for the same `ALDERLAKE_P`, different kernel versions request
    different firmware versions
    - 5.15 asks for 62.0.3
    - 5.18 asks for 69.0.3
    - 5.19 asks for 70.1.1
    - 6.1 asks for 70, then 70.1.1, and then 69.0.3

## Caching

- `i915_gem_object_pin_map` is called when the kernel needs to access the
  BO from CPU
  - e.g., cmd parsing, reloc patching
  - it calls vmap internally to set up a WB or WC mapping
  - this does not modify the kernel linear map
- `i915_gem_mmap_ioctl` is called when ther userspace needs to access the BO
  from CPU
  - it calls `vm_mmap` to set up a vma for the shmem file, and modifies
    `vma->vm_page_prot` to use the desired cache mode

## IRQs

- `gen8_de_irq_handler` for display engine related interrupts
  - `GEN8_PIPE_VBLANK` is generated on vblank, if anyone holds on to vblank
    with `drm_vblank_get`
- `gen8_gt_irq_handler` for graphics related interrupts
  - `GT_RENDER_USER_INTERRUPT` is generated in response to `MI_USER_INTERRUPT` cmd
  - `GT_CONTEXT_SWITCH_INTERRUPT` is generated on context switch
    - always on
    - programmed to be generated at the end of each request
    - requests are submitted to the HW on this interrupt.
    - when GPU is idle, it runs the idle context.  A single request causes
      two context switches: idle->req->idle

## Old

aperture
- i915_probe_agp returns aperture and preallocated sizes.
  the whole memory is called aperture.  the beginning some memory is stolen.
  it consists of gtt and preallocated memories.
- two drm_mm is created to manage preallocated memory and the reset of the aperture.
- drm_mm does not touch real memory, only offsets and sizes, which makes it generic

Memory Domains
- CPU: cpu addr -> mmu -> system ram
- GTT: cpu addr -> agp -> gpu addr -> gart -> system ram
- GPU: ~(CPU | GTT)
- GTT space is limited, comparing to CPU space
- there is always one write domain
- there may be multiple read domains

drm_gem_object
- DRM_I915_GEM_CREATE creates drm_gem_object through drm_gem_object_alloc.
- the object is shmem-based and a handle is added
- i915_gem_object_get_page_list checks the backing memory pointer and
  constructs a page_list for the physical pages of the memory.
- i915_gem_object_bind_to_gtt finds a free space in aperture and binds the
  backing pages of the object there.
- pin is almost equal to bind gtt.
- active object cannot be unbound.  bound or pinned object may be unmanaged, i.e. not on any list.
  when a request is retired, it is moved either to flushing or inactive list.
- mmap gtt creates a fake offset to allow userspace to mmap this object in
  aperture.  It binds the object.  The domain should be GTT
- mmap (non-gtt) maps the backing shmem of the object.  The difference from gtt
  mapping is that accessing gtt mapped region goes through agp and it makes
  linearly accessing tiled buffer possible.
- intel bufmgr drm_intel_gem_bo_map calls DRM_IOCTL_I915_GEM_MMAP to
  mmap the backing shmem and move the object to CPU domain.
  It is for swrast.
- glamo for example, bind/unbind moves object to/from vram.  set_domain flushes
  cpu caches or glamo rendering.
- gem on aperture may be tiled.  to make gtt access seems linear, fence regs
  must be set up.

lifetime
- an object is active if it is on active or flushing list.  active object
  cannot be unbound.
- when a pinned object is done, it becomes unmanaged because it cannot be
  evicted.
- an object on active list with write_domain will be moved to flushing list
  when done.  one without write_domain will be moved to inactive list.
- i915_add_request creates a seqno and installs an interrupt.  It removes
  the write domain of objects on flushing list with the specified flushing domains
  and moves them to active list.
- i915_wait_request waits until the seqno passes the given one.
- i915_gem_retire_request retires the first object(s) matching req->seqno by
  moving them to flushing or inactive list depending on their write_domain.
- i915_gem_retire_requests retires all passed requests.
- i915_gem_flush invalidates and/or flushes the given domains by putting the
  commands on the ring.
- i915_gem_evict_something is called when there is no aperture.  If there is
  inactive object, it unbinds the object.  Otherwise, it waits for a request.
  to finish, hoping it will make some room.  If there is no request, the first
  object on flushing list is flushed.  A requets is made and waited in that case.

mmap
- when /dev/drm/card? is mmaped, drm_gem_mmap is called
- if the region has a fake offset (GTT mapped), gem_vm_ops is installed
- otherwise, drm_mmap is called.
- if there is no offset and no agp, drm_mmap_dma is called and drm_vm_dma_ops is installed.
- otherwise, REGISTERS/FRAME_BUFFER/... is mapped
- it is used by drmAddMap and drmMap

PIO
- when pread, i915_gem_object_set_cpu_read_domain_range is called to move
  intetested range to CPU domain (not entirely valid).
  For write domain, i915_gem_object_flush_gpu_write_domain and
  i915_gem_object_wait_rendering are called so that there is no
  unexpcted writes.  After then, drm_clflush_pages is called to invalidate
  CPU caches.  There seems to be a missing drm_agp_chipset_flush which should
  be called after clflush??
-

/dev/dri/cardX
- Given a pci device and a type (always known by its driver, see drm_get_dev), 
  a new minor_id and a drm_minor is associated.  An dev node is created with
  the minor.
- every cardX is a drm_minor; every open creates a drm_file as file private.
  A drm_master for the drm_minor is created if none exists, and the drm_file
  becomes a master and is automatically authenticated.
- open a cardX -> drm_stub_open -> install and use driver fops, which usually
  has drm_open as its .open

modeset
- the drm_device contains mode_config, which is a drm_mode_config
- drm_mode_config has, among others, a list of crtcs, a list of connectors, and a list of encoders
- connectors have an encoder and modes, which are the modes the connected display supports
  crtc has a current mode and a desired mode
  The relation is (my guess): card -> crtcs -> routing encoders -> connectors (connector has no encoder if not routed)
- connector has encoder id; encoder has crtc id; crtc has buffer id
- after deciding the desired mode, userspace creates a bo, ADDFB, and SETCRTC.
  ADDFB creates intel_framebuffer around the bo.
  SETCRTC sets the specified crtc.  This includes the scan-out fb and the mode of the crtc,
	and the connectors connected to it.  As for the latter, it might involve move connectors
	from other crtc to this one or disconnect some connectors from this crtc
- i915 resources on eeepc

        Resources
        
        count_connectors : 3
        count_encoders   : 3
        count_crtcs      : 2
        count_fbs        : 0
        
        Connector: 1-1
                id             : 5
                encoder id     : 0
                conn           : unknown
                size           : 0x0 (mm)
                count_modes    : 0
                count_props    : 2
                props          : 1 2
                count_encoders : 1
                encoders       : 6
        Connector: 7-1
                id             : 7
                encoder id     : 8
                conn           : connected
                size           : 0x0 (mm)
                count_modes    : 1
                count_props    : 2
                props          : 1 2
                count_encoders : 1
                encoders       : 8
        Mode: "1024x600" 1024x600 60
        Connector: 6-1
                id             : 10
                encoder id     : 0
                conn           : disconnected
                size           : 0x0 (mm)
                count_modes    : 0
                count_props    : 7
                props          : 1 2 18 14 16 15 17
                count_encoders : 1
                encoders       : 11
        
        Encoder
                id     :6
                crtc_id   :0
                type   :1
                possible_crtcs  :3
                possible_clones :1
        Encoder
                id     :8
                crtc_id   :4
                type   :3
                possible_crtcs  :2
                possible_clones :2
        Encoder
                id     :11
                crtc_id   :0
                type   :4
                possible_crtcs  :3
                possible_clones :4
        
        Crtc
                id             : 3
                x              : 0
                y              : 0
                width          : 0
                height         : 0
                mode           : 0x804b4c4
                gamma size     : 256
        Crtc
                id             : 4
                x              : 0
                y              : 0
                width          : 0
                height         : 0
                mode           : 0x804b4c4
                gamma size     : 256


