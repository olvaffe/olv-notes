Mesa
====

## People

* Dave Airlie (redhat, kernel drm maintainer, radeon-rewrite)
* Eric Anholt (intel, kernel drm i915 maintainer?)
* Jesse Barnes (intel, libdrm maintainer)
* Kristian Høgsberg (redhat, dri2)
* Ian Romanick (intel)
* Keith Packard (intel)
* Keith Whitwell (vmware, acquired tungstengraphics)

## EXT and ARB

* EXT are extensions
* ARB are EXT reviewed and approved by ARB

## Overview

* <http://olvaffe.blogspot.com/2008/12/dri.html>
* glxgears: `i915_dri.so` and `/dev/dri/card0` is mapped, self busy
* with `LIBGL_ALWAYS_INDIRECT=1`, no such mappings and xorg is busy
* @kernel: DRM driver
* @libdrm: provides easy access to DRM (drm.h), and util functions for xorg impl.
*          (xf86drm.h).
* @`i915_dri.so`: 3D OpenGL driver
* @x11drv: 2D video driver, complicated by Xv, XvMC, resource sharing with 3d driver, etc.
* @x11: GLX and DRI extension. video driver for 2D, GLX for 3d, using `i915_dri.so` for AIGLX.
* @libgl: (over GLX) Use X DRI and GLX extention to implement glX*.  Use `i915_dri.so` for rendering.
* @app: call libglx then direct rendering or indirect one automagically


## Mesa

* Directories

        libGLU.so:
        src/glu: GLU
        
        libGLUT.so
        src/glut: GLUT, freeglut is prefered.
        
        libGL.so (Win)
        src/glw: OpenGL/Windows
        
        libGL.so (X11)
        src/glx/mini: Subset of GLX API, drawing directly to fb.
        src/glx/x11: OpenGL/X11
        
        libEGL.so
        src/egl/main: EGL, another dir for its driver, libEGLdri.so
        
        mesa: SW and various drivers implementing OpenGL API

## Mesa Defined

* glx/miniglx provides libGL.so which implements MiniGLX using DRI.
  The API and the underlying window system (fb0) are defined by mesa.
* mesa/drivers/osmesa provides libOSMesa.so which implements OSMesa.
  It provides software rendering to offscreen buffers.
* mesa/drivers/fbdev provides libGL.so which implements FBDev.
  It provides software rendering to fb0.

## Build

* choose and use one of `configs/`
* ./configure generates `configs/autoconf` from `configs/autoconf.in`
* if not building loader (`libGL`, etc.), touch `src/mesa/libglapi.a`
* config
 
       CFLAGS:          -g -O2 -Wall -Wmissing-prototypes -std=c99 -ffast-math -fno-strict-aliasing -fPIC
       CXXFLAGS:        -g -O2 -Wall -fno-strict-aliasing -fPIC
       Macros:          -D_GNU_SOURCE -DPTHREADS -DHAVE_POSIX_MEMALIGN -DUSE_EXTERNAL_DXTN_LIB=1 -DIN_DRI_DRIVER -DGLX_DIRECT_RENDERING -DGLX_INDIRECT_RENDERING -DHAVE_ALIAS -DUSE_X86_ASM -DUSE_MMX_ASM -DUSE_3DNOW_ASM -DUSE_SSE_ASM
* build swrast_dri.so

        /bin/sh ../../../../../bin/mklib -o swrast_dri.so -noprefix -linker
        'gcc' -ldflags '' ../../common/driverfuncs.o ../common/utils.o swrast.o
        swrast_span.o    ../../../../../src/mesa/libmesa.a -lm -lpthread -lexpat
        -ldl -ldrm

## make linux

* no dri, no x86 asm
* it gives `libGL.so` based on `driver/x11`, which is fake glx.  CPU is eaten by
  client.
* it gives `libEGL.so`.
* it gives `egl_softpipe.so`.  It links to `src/mesa/libmesagallium.a` and
  `src/mesa/libglapi.a`.  It is an EGL driver and it expects X display to be
  passed when `eglGetDisplay`.  Its native window system is X.
* it gives `gallium/libGL.so`.  It links to `libxlib.a` which is a fake glx.  It
  links to `libmesagallium.a`, etc, and supports several pipe drivers.
* chdir to `gallium/state_trackers/es` and make to give `libGLESv1_CM.so` and
  `libGLESv2.so`.

## progs

* `egl` is for use with `libEGL.so` and `EGL_i915.so` from gallium.
* `xdemos` is for use with real/fake glx `libGL.so`
* `demos` is for use with `libglut.so` (over glx `libGL.so`)
* `es1/xegl` is for use with `libEGL.so`, `libGLESv1_CM.so`, and
  `egl_softpipe.so`.
  * how to use them with `gallium/libGL.so`?
  * declare the prototype of `glFrustum`!
  * otherwise, float is passed while double is expected
  * `movss` is used for float and `movsd` is used for double.  `libGL.so` uses
    `movsd` is `glFrustum`. But without given a prototype, program might think
    it should pass float, and it uses `movss`.
