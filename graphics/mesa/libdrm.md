libdrm
======

## Core API

- enumeration functions
  - `drmGetDevices2` and `drmGetDevices` enumerate available drm devices
    - they scan `/dev/dri`, stat for `dev_t`, and parse `/sys` for device info
  - `drmFreeDevices` and `drmFreeDevice` free `drmDevicePtr`
- `dev_t` (dev major/minor) functions
  - `drmGetDeviceFromDevId` returns `drmDevicePtr` from `dev_t`
    - it pretty much `drmGetDevices2` and filters by `dev_t`
  - `drmGetNodeTypeFromDevId` returns the node type (primary or render) of a
    `dev_t`
- query functions
  - `drmGetDevice2` and `drmGetDevice` returns `drmDevicePtr` from an fd
    - they `fstat` the fd to get `dev_t` and call `drmGetDeviceFromDevId`
  - `drmDevicesEqual` returns true if two `drmDevicePtr` point to the same drm
    device
    - for pci devices, it compares pci domain/bus/dev/func
  - `drmGetVersion` wraps `DRM_IOCTL_VERSION`
    - it queries kernel driver version and name
  - `drmFreeVersion` wraps `free`
  - `drmGetCap` wraps `DRM_IOCTL_GET_CAP`
    - it queries a driver cap such as prime, syncobj, dumb, modifier, etc.
  - `drmGetNodeTypeFromFd` returns the node type (primary or render) of an fd
  - `drmGetDeviceNameFromFd` returns the primary node path for fd
    - `drmGetPrimaryDeviceNameFromFd` is the newer replacement
  - `drmGetDeviceNameFromFd2` returns the node path for fd
  - `drmGetPrimaryDeviceNameFromFd` returns the primary node path for fd
  - `drmGetRenderDeviceNameFromFd` returns the render node path for fd
- prime functions
  - `drmPrimeFDToHandle` wraps `DRM_IOCTL_PRIME_FD_TO_HANDLE`
  - `drmPrimeHandleToFD` wraps `DRM_IOCTL_PRIME_HANDLE_TO_FD`
  - `drmCloseBufferHandle` wraps `DRM_IOCTL_GEM_CLOSE`
- syncobj functions
  - `drmSyncobjCreate` wraps `DRM_IOCTL_SYNCOBJ_CREATE`
  - `drmSyncobjDestroy` wraps `DRM_IOCTL_SYNCOBJ_DESTROY`
  - `drmSyncobjReset` wraps `DRM_IOCTL_SYNCOBJ_RESET`
  - `drmSyncobjSignal` wraps `DRM_IOCTL_SYNCOBJ_SIGNAL`
  - `drmSyncobjWait` wraps `DRM_IOCTL_SYNCOBJ_WAIT`
  - `drmSyncobjTimelineSignal` wraps `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL`
  - `drmSyncobjTimelineWait` wraps `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT`
  - `drmSyncobjTransfer` wraps `DRM_IOCTL_SYNCOBJ_TRANSFER`
  - `drmSyncobjQuery2` and `drmSyncobjQuery` wrap `DRM_IOCTL_SYNCOBJ_QUERY`
  - `drmSyncobjEventfd` wraps `DRM_IOCTL_SYNCOBJ_EVENTFD`
  - `drmSyncobjFDToHandle` and `drmSyncobjImportSyncFile` wrap
    `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE`
  - `drmSyncobjHandleToFD` and `drmSyncobjExportSyncFile` wrap
    `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD`
- misc functions
  - `drmIoctl` wraps `ioctl`
  - `drmCommandNone`, `drmCommandRead`, `drmCommandWrite`, and
    `drmCommandWriteRead` wrap `drmIoctl`
  - `drmGetFormatName` returns the fourcc string of a fourcc format
  - `drmGetFormatModifierName` returns the descriptive name of a modifier
  - `drmGetFormatModifierVendor` returns the vendor name of a modifier
- legacy misc functions
  - `drmGetLibVersion` returns libdrm version
  - `drmAvailable` returns true if it can open the first primary node
    - this is still used by xserver DRI1
  - `drmMsg` wraps `vfprintf`
  - `drmMalloc` wraps `calloc`
  - `drmFree` wraps `free`
  - `drmMap` wraps `mmap`
  - `drmUnmap` wraps `munmap`
- legacy open functions
  - modern way is to enumerate and `open`/`close` directly instead
  - `drmOpenWithType` opens by driver name (or busid)
    - this is still used by some mesa tools
  - `drmOpen` wraps `drmOpenWithType` to open a primary node
    - this is still used by xserver DRI1 and modesetting
  - `drmOpenRender` opens a render node by its minor number
  - `drmOpenControl` opens a control node by its minor number
    - kernel never creates control nodes
  - `drmClose` updates global `drmHashTable` before `close`
    - this is still used by xserver DRI1 and modesetting
  - `drmOpenOnceWithType` opens by busid
    - if the same busid is opened multiple times, the same fd is returned
  - `drmOpenOnce` wraps `drmOpenOnceWithType` to open a primary node
  - `drmCloseOnce` wraps `drmClose` when the last ref to fd is closed

## Primary Node API

- negotiation functions
  - `drmIsKMS` calls `DRM_IOCTL_MODE_GETRESOURCES` to see if modeset is
    supported
  - `drmSetClientCap` wraps `DRM_IOCTL_SET_CLIENT_CAP`
    - modeset clients use this to enable atomic modesetting, universal planes,
      etc.
- master functions
  - `drmIsMaster` returns true if the `drm_file` is master
  - `drmSetMaster` wraps `DRM_IOCTL_SET_MASTER`
  - `drmDropMaster` wraps `DRM_IOCTL_DROP_MASTER`
- lease functions
  - `drmModeCreateLease` wraps `DRM_IOCTL_MODE_CREATE_LEASE`
  - `drmModeRevokeLease` wraps `DRM_IOCTL_MODE_REVOKE_LEASE`
  - `drmModeGetLease` wraps `DRM_IOCTL_MODE_GET_LEASE`
  - `drmModeListLessees` wraps `DRM_IOCTL_MODE_LIST_LESSEES`
- auth functions
  - this is necessary when using the primary node for rendering
  - `drmGetMagic` wraps `DRM_IOCTL_GET_MAGIC`
    - this queries the magic of a primary `drm_file`
    - the magic can be sent to master for auth
  - `drmAuthMagic` wraps `DRM_IOCTL_AUTH_MAGIC`
  - `drmGetClient` wraps `DRM_IOCTL_GET_CLIENT`
    - it returns whether the `drm_file` is authenticated
- resource functions
  - `drmModeGetResources` wraps `DRM_IOCTL_MODE_GETRESOURCES`
    - it returns resource ids for fbs, crtcs, connectors, and encoders
  - `drmModeFreeResources`
  - `drmModeGetFB` wraps `DRM_IOCTL_MODE_GETFB`
    - it queries fb info from a fb id
  - `drmModeFreeFB`
  - `drmModeGetFB2` wraps `DRM_IOCTL_MODE_GETFB2`
  - `drmModeFreeFB2`
  - `drmModeGetCrtc` wraps `DRM_IOCTL_MODE_GETCRTC`
    - it queries crtc info from a crtc id
  - `drmModeFreeCrtc`
  - `drmModeGetConnector` and `drmModeGetConnectorCurrent` wrap
    `DRM_IOCTL_MODE_GETCONNECTOR`
    - it queries connector info from a connector id
    - `drmModeGetConnectorCurrent` limits the supported modes to the current
      mode
  - `drmModeFreeConnector`
  - `drmModeConnectorGetPossibleCrtcs`
    - it calls `drmModeGetEncoder` on all encoders of the connector
  - `drmModeGetEncoder` wraps `DRM_IOCTL_MODE_GETENCODER`
    - it queries encoder info from an encoder id
    - props are newer replacement
  - `drmModeFreeEncoder`
  - `drmModeGetPlaneResources` wraps `DRM_IOCTL_MODE_GETPLANERESOURCES`
    - it returns resource ids for planes
  - `drmModeFreePlaneResources`
  - `drmModeGetPlane` wraps `DRM_IOCTL_MODE_GETPLANE`
    - it queries plane info from a plane id
  - `drmModeFreePlane`
- property functions
  - `drmModeObjectGetProperties` wraps `DRM_IOCTL_MODE_OBJ_GETPROPERTIES`
    - it returns prop ids for a resource
  - `drmModeFreeObjectProperties`
  - `drmModeGetProperty` wraps `DRM_IOCTL_MODE_GETPROPERTY`
    - it queries prop info from a prop id
  - `drmModeFreeProperty`
  - `drmModeGetPropertyBlob` wraps `DRM_IOCTL_MODE_GETPROPBLOB`
    - it queries blob info from a blob id
  - `drmModeFreePropertyBlob`
  - `drmModeCreatePropertyBlob` wraps `DRM_IOCTL_MODE_CREATEPROPBLOB`
  - `drmModeDestroyPropertyBlob` wraps `DRM_IOCTL_MODE_DESTROYPROPBLOB`
- dumb buffer functions
  - `drmModeCreateDumbBuffer` wraps `DRM_IOCTL_MODE_CREATE_DUMB`
  - `drmModeDestroyDumbBuffer` wraps `DRM_IOCTL_MODE_DESTROY_DUMB`
  - `drmModeMapDumbBuffer` wraps `DRM_IOCTL_MODE_MAP_DUMB`
- fb functions
  - `drmModeAddFB` wraps `DRM_IOCTL_MODE_ADDFB`
  - `drmModeAddFB2` and `drmModeAddFB2WithModifiers` wrap
    `DRM_IOCTL_MODE_ADDFB2`
  - `drmModeRmFB` wraps `DRM_IOCTL_MODE_RMFB`
  - `drmModeCloseFB` wraps `DRM_IOCTL_MODE_CLOSEFB`
    - it removes the fb without disabling crtc/plane
  - `drmModeDirtyFB` wraps `DRM_IOCTL_MODE_DIRTYFB`
    - it is only used by xserver modesetting
- atomic functions
  - `drmModeAtomicAlloc`
  - `drmModeAtomicFree`
  - `drmModeAtomicDuplicate`
  - `drmModeAtomicMerge`
  - `drmModeAtomicSetCursor`
    - this is not mouse cursor, but the idx into the prop array
  - `drmModeAtomicGetCursor`
  - `drmModeAtomicAddProperty`
  - `drmModeAtomicCommit` wraps `DRM_IOCTL_MODE_ATOMIC`
- non-atomic functions
  - `drmModeObjectSetProperty` wraps `DRM_IOCTL_MODE_OBJ_SETPROPERTY`
  - `drmModeConnectorSetProperty` wraps `DRM_IOCTL_MODE_SETPROPERTY`
    - `DRM_IOCTL_MODE_OBJ_SETPROPERTY` is the newer replacement
  - `drmModeCrtcSetGamma` wraps `DRM_IOCTL_MODE_SETGAMMA`
    - blobs are newer replacement
  - `drmModeCrtcGetGamma` wraps `DRM_IOCTL_MODE_GETGAMMA`
  - `drmModeSetCrtc` wraps `DRM_IOCTL_MODE_SETCRTC`
  - `drmModeSetPlane` wraps `DRM_IOCTL_MODE_SETPLANE`
  - `drmModeSetCursor` and `drmModeMoveCursor` wrap `DRM_IOCTL_MODE_CURSOR`
  - `drmModeSetCursor2` wraps `DRM_IOCTL_MODE_CURSOR2`
  - `drmModePageFlip` and `drmModePageFlipTarget` wrap
    `DRM_IOCTL_MODE_PAGE_FLIP`
- vblank functions
  - `drmCrtcGetSequence` wraps `DRM_IOCTL_CRTC_GET_SEQUENCE`
  - `drmCrtcQueueSequence` wraps `DRM_IOCTL_CRTC_QUEUE_SEQUENCE`
  - `drmWaitVBlank` wraps `DRM_IOCTL_WAIT_VBLANK`
    - `DRM_IOCTL_CRTC_QUEUE_SEQUENCE` is the newer replacement
    - it is still used by xserver modesetting as fallback
  - `drmHandleEvent` reads a DRM event (vblank, flip, or crtc seq)
- misc functions
  - `drmModeGetConnectorTypeName` returns the connector type name
  - `drmModeFormatModifierBlobIterNext` parses the `IN_FORMATS` prop blob
- legacy mode functions
  - `drmModeAttachMode` wraps `DRM_IOCTL_MODE_ATTACHMODE`
    - it is stubbed out in kernel
  - `drmModeDetachMode` wraps `DRM_IOCTL_MODE_DETACHMODE`
    - it is stubbed out in kernel
  - `drmModeFreeModeInfo` is unused
- legacy busid functions
  - some are still used by xserver DRI1 and modesetting
  - `drmSetInterfaceVersion` wraps `DRM_IOCTL_SET_VERSION`
    - kernel sets the busid for the device
  - `drmSetBusid` wraps `DRM_IOCTL_SET_UNIQUE`
    - kernel always returns `-EINVAL`
  - `drmGetBusid` wraps `DRM_IOCTL_GET_UNIQUE`
    - it returns the busid if previously set
  - `drmFreeBusid` wraps `free`
  - `drmCheckModesettingSupported` returns true if busid supports modesetting

## Legacy DRI1 API

- legacy context functions
  - many are still used by xserver DRI1
  - `drmCreateContext` wraps `DRM_IOCTL_ADD_CTX`
  - `drmDestroyContext` wraps `DRM_IOCTL_RM_CTX`
  - `drmSetContextFlags` wraps `DRM_IOCTL_MOD_CTX`
  - `drmGetContextFlags` wraps `DRM_IOCTL_GET_CTX`
  - `drmSwitchToContext` wraps `DRM_IOCTL_SWITCH_CTX`
  - `drmGetReservedContextList` wraps `DRM_IOCTL_RES_CTX`
  - `drmFreeReservedContextList` wraps `free`
  - `drmAddContextPrivateMapping` wraps `DRM_IOCTL_SET_SAREA_CTX`
  - `drmGetContextPrivateMapping` wraps `DRM_IOCTL_GET_SAREA_CTX`
  - `drmAddContextTag`, `drmDelContextTag`, and `drmGetContextTag` manipulate
    the global `drmHashTable`
- legacy drawable functions
  - these are still used by xserver DRI1
  - `drmCreateDrawable` wraps `DRM_IOCTL_ADD_DRAW`
  - `drmDestroyDrawable` wraps `DRM_IOCTL_RM_DRAW`
  - `drmUpdateDrawableInfo` wraps `DRM_IOCTL_UPDATE_DRAW`
- legacy agp functions
  - `drmAgpAcquire` wraps `DRM_IOCTL_AGP_ACQUIRE`
  - `drmAgpAlloc` wraps `DRM_IOCTL_AGP_ALLOC`
  - `drmAgpBind` wraps `DRM_IOCTL_AGP_BIND`
  - `drmAgpEnable` wraps `DRM_IOCTL_AGP_ENABLE`
  - `drmAgpFree` wraps `DRM_IOCTL_AGP_FREE`
  - `drmAgpRelease` wraps `DRM_IOCTL_AGP_RELEASE`
  - `drmAgpUnbind` wraps `DRM_IOCTL_AGP_UNBIND`
  - `drmAgpBase`, `drmAgpDeviceId`, `drmAgpGetMode`, `drmAgpMemoryAvail`,
    `drmAgpMemoryUsed`, `drmAgpSize`, `drmAgpVendorId`, `drmAgpVersionMajor`,
    and `drmAgpVersionMinor` wrap `DRM_IOCTL_AGP_INFO`
- legacy dma functions
  - some are still used by xserver DRI1
  - `drmGetMap` wraps `DRM_IOCTL_GET_MAP`
  - `drmAddMap` wraps `DRM_IOCTL_ADD_MAP`
  - `drmRmMap` wraps `DRM_IOCTL_RM_MAP`
  - `drmAddBufs` wraps `DRM_IOCTL_ADD_BUFS`
  - `drmFreeBufs` wraps `DRM_IOCTL_FREE_BUFS`
  - `drmGetBufInfo` wraps `DRM_IOCTL_INFO_BUFS`
  - `drmMapBufs` wraps `DRM_IOCTL_MAP_BUFS`
  - `drmUnmapBufs` wraps `munmap`
  - `drmMarkBufs` wraps `DRM_IOCTL_MARK_BUFS`
  - `drmDMA` wraps `DRM_IOCTL_DMA`
- legacy irq functions
  - `drmCtlInstHandler` and `drmCtlUninstHandler` wrap `DRM_IOCTL_CONTROL`
  - `drmGetInterruptFromBusID` wraps `DRM_IOCTL_IRQ_BUSID`
- legacy lock functions
  - `drmGetLock` wraps `DRM_IOCTL_LOCK`
  - `drmUnlock` wraps `DRM_IOCTL_UNLOCK`
    - it is still used by xserver DRI1
- legacy scatter-gather functions
  - `drmScatterGatherAlloc` wraps `DRM_IOCTL_SG_ALLOC`
  - `drmScatterGatherFree` wraps `DRM_IOCTL_SG_FREE`
- legacy misc functions
  - `drmGetStats` wraps `DRM_IOCTL_GET_STATS`
    - it is stubbed out in kernel
  - `drmFinish` wraps `DRM_IOCTL_FINISH`
    - it is stubbed out in kernel
  - `drmSetServerInfo` sets `drm_server_info`
    - it is still used by xserver DRI1 to change permissions, load kernel
      modules, etc.
  - `drmError` returns the string of an drm error code
- legacy hash functions
  - many are still used by xserver DRI1
  - `drmGetHashTable` returns the global `drmHashTable`
  - `drmGetEntry` queries the hash entry associated with the fd
  - `drmHashCreate`
  - `drmHashDelete`
  - `drmHashDestroy`
  - `drmHashFirst`
  - `drmHashInsert`
  - `drmHashLookup`
  - `drmHashNext`
- legacy rng functions
  - `drmRandom`
  - `drmRandomCreate`
  - `drmRandomDestroy`
  - `drmRandomDouble`
- legacy skip list functions
  - `drmSLCreate`
  - `drmSLDelete`
  - `drmSLDestroy`
  - `drmSLDump`
  - `drmSLFirst`
  - `drmSLInsert`
  - `drmSLLookup`
  - `drmSLLookupNeighbors`
  - `drmSLNext`

## `drmdevice`

- it is to demonstrate `drmGetDevices2` and `drmGetDevice2`
  - it calls `drmGetDevices2` to enumerate all `drmDevicePtr` and dumps them
  - it then calls `drmGetDevice2` on each dev node and dumps the same info again
  - no root required, except opening primary nodes might require `video` group

## `vbltest`

- it is to demonstrate `drmWaitVBlank`
  - it times 60 vblanks and prints the vblank frequency
  - no root required, except opening primary nodes might require `video` group
- it uses the older `drmOpen(driver_name, bus_id)` to open the device
  - only one of `driver_name` and `bus_id` is needed
    - it tries `drmOpenByBusid` and then `drmOpenByName`
    - internally, they scan all devices and return the first device that
      matches the condition
  - modern way is to `drmGetDevices2` and `open` instead
- it uses `drmWaitVBlank` with `sequence = 0` and `type = DRM_VBLANK_RELATIVE`
  to get the current vblank count
- it uses `drmWaitVBlank` with `sequence = 1` and
  `type = DRM_VBLANK_RELATIVE | DRM_VBLANK_EVENT` to queue an event on next
  vblank
- when events are delivered, drm fd becomes readable
- `drmHandleEvent` can read events from drm fd
  
## `proptest`

- it is to demonstrate `drmModeGetResources`, `drmModeObjectGetProperties`,
  `drmModeGetProperty`, and `drmModeObjectSetProperty`
  - `drmModeGetResources` returns all mode objects: fbs, crtcs, connectors,
    and encoders
    - note that it only dumps crtcs and connectors but not other mode objects
  - each object has some properties that can be set/get
  - no root required, except opening primary nodes might require `video` group
- each property has
  - name
    - a connector has `EDID`, `DPMS`, etc.
    - a crtc has `VRR_ENABLED`, `CTM`, etc.
  - `flags` can be anything but some bits are used for property value types
    (enum, blob, range, etc.)
  - values

## `modeprint`

- it is to demonstrate `drmModeGetResources`
  - it dumps all resources by default
  - `-v` to further dump connector mode and property details
    - note that it only dumps property details of connectors but not other mode objects
  - no root required, except opening primary nodes might require `video` group

## `modetest`

- it can set/get everything
- when no argument is specified, it dumps all mode objects
  - `-e` to dump just encoders
  - `-c` to dump just connectors and their props
  - `-p` to dump just crtcs, planes, and their props
  - `-f` to dump just fbs
- otherwise, it sets mode objects
  - `-a` uses atomic modesetting
  - each `-w` sets one prop
  - each `-s` sets a connector to a mode
    - `-s 32:#0` sets connector 32 to its first mode
  - `-r` sets all connectors to their preferred modes
  - each `-P` sets a plane
    - `-P 41@65:1000x1000` assigns plane 41 to crtc 65.  The plane scans out
      from a 1000x1000 bo and is centered
  - `-F` sets the fb fill pattern
    - `smpte` is default
    - `tiles`
    - `plain`
    - `gradient`
- `-v` requires `-s` and tests vsyncs
  - it keeps flipping between two bos
  - it seems semi-broken without `-a`
  - if `-a`, it also requires `-P`
- `-C` requires `-s` and tests cursors
  - if `-a`, it is silently ignored

## Device Enumeration

- `drmGetDevices2` enumerates all DRM devices
- for each file under `/dev/dri`,
  - control nodes start with `controlD`
  - render nodes start with `renderD`
  - primary nodes start with `card`
- the function further checks `/sys/dev/char/<major>:<minor>/device`
  - `drm` subdirecotory indicates a DRM device
  - `subsystem` link indicates the bus (pci, usb, platform, virtio, etc)
  - for PCI device, `uevent`'s `PCI_SLOT_NAME` specifies the
    domain/bus/dev/func; other files specify
    revision/vendor/device/subvendor/subdevice ids
- all info about all DRM device are returned as `drmDevicePtr`s
- a driver usually inspects `drmDevicePtr` to determine if it is a supported
  device
  - the driver then opens the render node

## DRI Loader

- protocols such as DRI3 give the DRI loader an opened fd
- `drmGetDevice2` can return a `drmDevicePtr` for the fd
  - it works similar to `drmGetDevices2`, except it drops all but the only
    device having the matching <major>:<minor>
- the DRI loader uses the DRM device to pick the driver
  - it uses the PCI id and a built-in table to find the DRI driver name
  - it falls back to `drmGetVersion` to use the kernel driver name
  - it then loads `<name>_dri.so` and finds `__driDriverGetExtensions_<name>`
    symbol
- when prime is enabled, the DRI loader can intentionally open another render
  node as configured.

## DRI Driver Initialization

- the loader calls `createNewScreen2` with the opened fd from the DRI driver
- this is `driCreateNewScreen2` and calls `dri2_init_screen` for
  gallium-based DRI driver
- Gallium-based DRI driver is a mega dri driver: it consists of multiple
  drivers
  - the mega driver calls `pipe_loader_drm_probe_fd`
  - the function dups the fd, probes the fd for PCI id, and picks the
    corresponding `drm_driver_descriptor`
    - it dups because `pipe_loader_drm_release` closes
  - finally `pipe_loader_create_screen` calls
    `drm_driver_descriptor::create_screen` indirectly.  The callback function
    dups the fd again.
    - it dups because its `pipe_screen::destroy` closes
  - for some pipe drivers, two state trackers (vdpau and mesa, for example)
    must share the same `pipe_screen` to share resources.  ZaphodHeads allows
    multiple X screens from the same PCI device.  Combined, we dup fds.
