libdrm
======

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
