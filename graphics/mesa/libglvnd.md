libglvnd
========

## Overview

- glvnd provides these public headers
  - `/usr/include/{EGL,GL,GLES,GLES2,GLES3,KHR}`
    - these are mostly from Khronos directly
    - with the exception of `gl.h` and `glx.h`
  - `/usr/include/glvnd`
    - these define the apis that vendor icds must implement
- glvnd provides these public libraries
  - `/usr/lib/{libEGL.so.1,libGLESv1_CM.so.1,libGLESv2.so.2,libOpenGL.so.0}`
  - `/usr/lib/libGLX.so.0`
  - `/usr/lib/libGL.so.1`
  - `/usr/lib/libGLdispatch.so.0`
- glvnd provides these pkgconfigs
  - `/usr/lib/pkgconfig/{egl,glesv1_cm,glesv2,opengl}.pc`
  - `/usr/lib/pkgconfig/glx.pc`
  - `/usr/lib/pkgconfig/gl.pc`
  - `/usr/lib/pkgconfig/libglvnd.pc`

## `libEGL.so.1`

- `libEGL.so.1` exports 44 symbols
  - they are defined by EGL 1.5 (`grep ^EGLAPI egl.h`)
  - glvnd uses `PUBLIC` macro to export symbols
  - `gen_egl_dispatch.py` generates most of them based on the info provided by
    `eglFunctionList.py`
    - `_eglCore` functions that are not `custom` are generated
    - see the doc at the beginning of `eglFunctionList.py`
- `__eglInit` is executed when the library loads
  - it calls `__glDispatchInit` to initialize `libGLdispatch.so.0`
  - it initializes its internal states
- when any global function (any public symbol which does not need an
  `EGLDisplay`, such as `eglGetDisplay`) is called, `__eglLoadVendors` loads
  the vendor icds
  - `LoadVendors` scans `/etc/glvnd/egl_vendor.d` and
    `/usr/share/glvnd/egl_vendor.d` for the json files
  - `LoadVendorFromConfigFile` parses the json files
  - `LoadVendor` loads the icd and calls `__egl_Main` to exchange callbcks
    - `__EGLapiExports` are callbacks exported to icd
    - `__EGLapiImports` are callbacks imported from icd
  - `LookupVendorEntrypoints` initilizes `vendor->staticDispatch`
  - there are also `vendor->dynDispatch` and `glDispatch`
- when a regular public symbol, such as `eglCreateContext` is called,
  - it calls `__eglDispatchFetchByDisplay` to get the vendor icd function
  - `getVendorFromDisplay` calls `__eglGetVendorFromDisplay` to get the vendor
  - `FetchVendorFunc` calls `__eglFetchDispatchEntry` to get the vendor
    function
  - this seems slow
- when `eglMakeCurrent` is called,
  - it calls `__glDispatchMakeCurrent` to make `vendor->glDispatch` current

## `libGLX.so.0`

- `libGLX.so.0` exports 42 symbols
  - 39 of them are defined by GLX 1.4
  - `glXGetProcAddressARB` for legacy reason
  - `__GLXGL_CORE_FUNCTIONS` and `__glXGLLoadGLXFunction` are exposed to
    `libGL.so.1`
