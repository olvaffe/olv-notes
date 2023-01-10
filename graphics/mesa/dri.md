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
  - `loader_open_driver` dlopens `<name>_dri.so`, calls
    `__driDriverGetExtensions_<name>`, and returns an array of
    `__DRIextension *`
- GBM
  - `gbm_create_device`
  - `_gbm_create_device`
  - `find_backend`
  - `dri_device_create`
  - `dri_screen_create`, where `loader_get_driver_for_fd` is called on the
    user-provided fd
  - `dri_screen_create_for_driver`
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
- note that swrast (or wayland) uses different paths

## DRI driver extensions

- driver extensions
  - `loader_bind_extensions` is called after `loader_open_driver`
    - the loader (GLX, GBM, or EGL) provides a list of required and optional
      extensions
    - the function matches against the list of extensions returned by the DRI
      driver
  - GLX asks for `exts`
    - `__DRI_CORE`
    - `__DRI_IMAGE_DRIVER`
    - `__DRI_MESA`
  - GBM asks for `gbm_dri_device_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_DRI2`
  - EGL asks for `dri3_driver_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_IMAGE_DRIVER`
    - `__DRI_CONFIG_OPTIONS`
- loader extensions
  - loader provides a list of extensions to the driver in `createNewScreen`
  - GLX provides `loader_extensions`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_BACKGROUND_CALLABLE`
  - GBM provides `gbm_dri_screen_extensions`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_DRI2_LOADER`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_KOPPER_LOADER`
  - EGL provides `dri3_image_loader_extensions`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_BACKGROUND_CALLABLE`
- screen extensions
  - after a screen is created, `__DRIcoreExtensionRec::getExtensions` queries
    the screen extensions
  - GLX asks for `exts`
    - `__DRI2_RENDERER_QUERY`
    - `__DRI2_FLUSH`
    - `__DRI_IMAGE`
    - `__DRI2_INTEROP`
    - `__DRI2_CONFIG_QUERY`
  - GBM asks for `dri_core_extensions`
    - `__DRI2_FLUSH`
    - `__DRI_IMAGE`
  - EGL asks for `dri2_core_extensions` and `optional_core_extensions`
    - `__DRI2_FLUSH`
    - `__DRI_TEX_BUFFER`
    - `__DRI_IMAGE`
    - `__DRI2_CONFIG_QUERY`
    - `__DRI2_FENCE`
    - `__DRI2_BUFFER_DAMAGE`
    - `__DRI2_INTEROP`
    - `__DRI_IMAGE`
    - `__DRI2_FLUSH_CONTROL`
    - `__DRI2_BLOB`
    - `__DRI_MUTABLE_RENDER_BUFFER_DRIVER`
    - `__DRI_KOPPER`
- note that swrast uses different extensions

## Gallium DRI Megadriver

- `src/gallium/targets/dri/target.c` defines `__driDriverGetExtensions_<name>`
  for various drivers.  They return one of the following lists of extensions
  - `galliumdrm_driver_extensions` for hw drivers
  - `galliumsw_driver_extensions` for swrast
  - `dri_swrast_kms_driver_extensions` for `kms_swrast`
  - `galliumvk_driver_extensions` for zink
- to create the `pipe_screen`,
  - `driCreateNewScreen2`
  - `dri2_init_screen`
    - `pipe_loader_drm_probe_fd` is called to dup the fd, probe, and determine
      the `drm_driver_descriptor`
      - `loader_get_driver_for_fd` queries the driver name
      - `get_driver_descriptor` looks up `drm_driver_descriptor` from a
        statically-defined `driver_descriptors` array
        - the descriptors are defined via `DRM_DRIVER_DESCRIPTOR` 
      - it also sets `pipe_loader_ops` to `pipe_loader_drm_ops`
    - `pipe_loader_create_screen` creates the pipe screen
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

## swrast

- GLX
  - driver loading
    - `__glXInitialize`
    - `driswCreateDisplay`
    - `driswCreateScreen`
    - `driswCreateScreenDriver` (`swrast` or `zink`)
    - `driOpenDriver`
    - `loader_open_driver`
  - asked driver extensions, `exts`
    - `__DRI_CORE`
    - `__DRI_SWRAST`
    - `__DRI_KOPPER`
    - `__DRI_COPY_SUB_BUFFER`
    - `__DRI_MESA`
  - provided loader extensions, `loader_extensions_shm`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_KOPPER_LOADER`
  - asked screen extensions, `exts`
    - `__DRI_TEX_BUFFER`
    - `__DRI2_RENDERER_QUERY`
    - `__DRI2_FLUSH`
    - `__DRI2_CONFIG_QUERY`
- GBM
  - driver loading
    - `dri_device_create`
    - `dri_screen_create_sw` (try `zink`, `kms_swrast`, and `swrast` in order)
    - `dri_screen_create_for_driver`
    - `dri_open_driver`
    - `loader_open_driver`
  - asked driver extensions, `gbm_swrast_device_extensions`
    - same as hw
  - provided loader extensions, `gbm_dri_screen_extensions`
    - same as hw
  - asked screen extensions, none
- EGL
  - driver loading
    - `eglInitialize`
    - `dri2_initialize`
    - `dri2_initialize_x11`
    - `dri2_initialize_x11_swrast` (`zink` or `swrast`)
    - `dri2_load_driver_swrast`
    - `dri2_load_driver_common`
    - `dri2_open_driver`
    - `loader_open_driver`
  - asked driver extensions, `swrast_driver_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_SWRAST`
    - `__DRI_CONFIG_OPTIONS`
  - provided loader extensions, `swrast_loader_extensions`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_KOPPER_LOADER`
  - asked screen extensions, `swrast_core_extensions` and
    `optional_core_extensions`
    - `__DRI_TEX_BUFFER`
    - and optional ones
- swrast
  - the gallium dri megadriver provides `galliumsw_driver_extensions`, used by
    `swrast_dri.so`
- `kms_swrast`
  - the gallium dri megadriver provides  `dri_swrast_kms_driver_extensions`,
    used by `kms_swrast_dri.so`
  - this is preferred and is faster than `swrast_dri.so` by rendering to dumb
    bos directly (as opposed to shms)
    - this is also used with KMS-only kernel drivers, where mesa uses vgmem
