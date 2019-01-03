EGL has display, surface, context
DRI has screen, drawable, context

Flow
* eglCreateDisplayNative to create a display
  a EGLDisplay talks to dri driver to create a dri2 __DRIscreen
  screen configs and extensions are also queried.
  loader extentions are passed to dri driver
  Make sure there is __DRI_TEX_BUFFER and __DRI_COPY_BUFFER extensions
* eglInitialize is trivial
* eglChooseConfig to choose one (or more) configs matching the attr list.
  matching RGB888 and caveat == none in this case
* eglCreateSurfaceForName to create a EGLSurface with the gem buffer as front __DRIbuffer
  dri2 createNewDrawable is called
  attr specifies back buffer as render buffer, which makes loader never
    return front buffer to dri driver, intead always return newly created one
* eglCreateContext calls directly to createNewContext to create a __DRIcontext
* eglMakeCurrent to make the context and surface current
  which is trivial call to dri bindContext
* can start rendering now, and eglTerminate is called after rendering
  it is a noop for now
  i guess it should destory the surfaces, which involves gem closing the buffers
* eglSwapBuffers is called during rendering to show to the screen
  if not rendering to back buffer, it is a noop
  otherwise, call copy buffer extension to copy __DRI_BUFFER_FRONT_LEFT buffer
    to the real front buffer, as passed in in eglCreateSurfaceForName.
  

eglCreateDisplayNative creates a display
* it takes a struct udev_device, specifying the hardware device
* pci id is looked up, corresponding function, say i915CreateDisplay, is called
* eglInitDisplay is called and display->backend = &intelBackend;
* in eglInitDisplay, ...
* /dev/dri/card0 is opened
* invokes eglLoadDriver to dlopen /usr/lib/dri/i915_dri.so, and set display->core and display->dri2
  (see mesa/src/mesa/drivers/dri/common/dri_util.c)
* a dri screen is created at display->driScreen
* extentions are inited to display->texBuffer and display->copyBuffer
* dri configs are copied into display->configs

get_fb_name DRM modesetting
fd -> resources -> crtc -> fb -> flink.handle -> flink.name

eglCreateSurfaceForName creates a surface over fb
surface = {
	.backBuffer = TRUE,
	.front.name = name
	.frontHandle = 0,
	.display = display
	.width
	.height
	.driDrawable = display->dri2->createNewDrawable()
};

context = eglCreateContext(display, config, NULL, NULL);
eglMakeCurrent(display, surface, surface, context)
eglSwapBuffers(display, surface) { display->backend->swapBuffers(display, surface); }
eglTerminate(display);

In i915_dri.so, or dri driver in general, the .so interface is defined in common/dri_util.c

When create context, intelCreateContext is called, functions are inited in i915InitDriverFunctions, dispatch_table inited in intelInitContext (_mesa_initialize_context, _mesa_init_exec_table, _vbo_CreateContext, _tnl_CreateContext)
When make current, intelMakeCurrent is called
