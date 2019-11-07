libdrm
======

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
