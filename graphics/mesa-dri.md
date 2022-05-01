Mesa and DRI
============

## DRI driver loading

- GLX
  - `__glXInitialize`
  - `AllocAndFetchScreenConfigs`
  - `dri3_create_screen`
    - `loader_dri3_open` is called to get the fd from the server
    - `loader_get_user_preferred_fd` potentially picks another fd when
      `DRI_PRIME` is set or driconf is configured
    - `loader_get_driver_for_fd` returns the driver name using pci id mapping,
      kernel driver name, or `MESA_LOADER_DRIVER_OVERRIDE`
  - `driOpenDriver`
  - `loader_open_driver`
- GBM
  - `gbm_create_device`
  - `_gbm_create_device`
  - `dri_device_create`
  - `dri_screen_create`, where `loader_get_driver_for_fd` is called on the
    user-provided fd
  - `dri_screen_create_dri2`
  - `dri_load_driver`
  - `dri_open_driver`
  - `loader_open_driver`
- EGL
  - `eglInitialize`
  - `dri2_initialize`
  - `dri2_initialize_x11`
    - `dri2_initialize_x11_dri3`, where `loader_dri3_open`,
      `loader_get_user_preferred_fd`, and `loader_get_driver_for_fd` are
      called
    - `dri2_load_driver_dri3`
    - `dri2_load_driver_common`
    - `dri2_open_driver`
    - `loader_open_driver`
  - or, `dri2_initialize_drm`
    - `gbm_create_device`

## DRI driver extensions

- negotiations
  - `loader_open_driver` returns an array of driver extensions
  - loaders (GLX/GBM/EGL) checks if the DRI driver provides all driver
    extensions required
  - loaders calls `createNewScreen2` with loader extensions and driver
    extensions, to negotiate extensions and to create a screen
- modern DRI drivers provide `driImageDriverExtension`

## Gallium DRI Megadriver

- A megadriver has `megadriver_stub_init` as a static constructor
  - it initializes the driver extension array based on the filename
  - in `gallium/targets/dri`, it uses `DEFINE_LOADER_DRM_ENTRYPOINT` to define
    the sub-driver function to be called by `megadriver_stub_init`, named
    `__driDriverGetExtensions_foo`
- for gallium dri megadriver,
  - `galliumdrm_driver_extensions` is the driver extension array
  - `galliumdrm_driver_api` is the driver api used by the common DRI interface
- to create the `pipe_screen`,
  - `driCreateNewScreen2`
  - `dri2_init_screen`
    - `pipe_loader_drm_probe_fd` is called to dup the fd, probe, and determine
      the `drm_driver_descriptor`
    - `drm_driver_descriptor`s are defined via `DRM_DRIVER_DESCRIPTOR` 
    - it also sets `pipe_loader_ops` to `pipe_loader_drm_ops`
  - `pipe_loader_create_screen`
  - `pipe_loader_drm_create_screen`
  - `pipe_iris_create_screen`
  - `iris_drm_screen_create`
  - `iris_screen_create`
- kmsro
  - kmsro is used when the display uses one DRM driver and the GPU uses
    another DRM driver (e.g., mobile)
  - in `pipe_loader_drm_probe_fd`, we might fall back to
    `kmsro_driver_descriptor`
  - `kmsro_drm_screen_create` calls the gallium driver's screen create
    function with `struct renderonly`
- swrast
  - the gallium dri megadriver provides `galliumsw_driver_extensions` and
    `galliumsw_driver_api`, used by `swrast_dri.so`
  - it also provides `dri_kms_driver_api`, used by `kms_swrast_dri.so`
    - seems to be paired with KMS-only kernel drivers
