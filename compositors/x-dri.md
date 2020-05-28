X DRI Extension
===============

## History

- <https://dri.freedesktop.org/wiki/DriHistory/>
- released in 1998
- requires kernel `CONFIG_DRM_LEGACY` which is disabled by default

## Model

- the hw only has a front buffer and a back buffer (and depth/stencil buffer)
- kernel, server, and client share the same SAREA
- a client gains exclusive access by locking a mutex in SAREA
  - it renders to the back buffer
  - cliprects are used to avoid overwriting another client's contents
- front/back buffers are swapped for display

## Initialization

- `XF86DRIGetClientDriverName`
  - server returns the driver name
  - client loads the driver (`/usr/lib/dri/${name}_dri.so`)
- `XF86DRIOpenConnection`
  - server returns SAREA address and bus id
  - client opens the DRI device
    - `drmOpenOnce` finds the first primary device (`/dev/dri/cardX`) whose
      `drmGetBusid` returns a compatible bus id
  - client mmaps SAREA
- `XF86DRIAuthConnection`
  - client gets the magic using `drmGetMagic`
  - server authenticates the magic, which enables client to do more ioctls
- `XF86DRIGetDeviceInfo`
  - server returns fb address and size
  - client mmaps fb
  - a driver screen is created from the fb and SAREA

## Contexts and Drawables

- `XF86DRICreateContext`
  - server returns a context
  - client creates a driver context from the server context
- `XF86DRICreateDrawable`
  - server returns a drawable
  - client creates a driver drawable from the server drawable
- `XF86DRIGetDrawableInfo`
  - server returns drawable x/y/w/h/cliprects
  - requested by client driver
