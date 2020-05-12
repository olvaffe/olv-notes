Enlightenment
=============

## ecore

- Every top-level visible window is an `Evas`.
- A top-level-visible window and the associated `Evas` are created by ecore.
- window events or user inputs are also handled by ecore
- ecore runs a mainloop and events like user inputs or resize are fed to evas through
  `evas_output_size_set` or `evas_event_feed_key_down`, or etc.

## evas

- ecore will create an `Evas` and pass an engine-specific info (like drawable
  id, and etc.) to it through `evas_engine_info_set`.
- engine will initialize itself, incuding an engine output and engine context.
  - an engine output consists of an EGLDisplay, EGLSurface, and an EGLContext.
    It also tracks the rendering state such as the vertices, and etc.
  - an engine context is usually a generic `RGBA_Draw_Context`.
- The prototype of object rendering is
  `void (*render) (Evas_Object *obj, void *output, void *context, void *surface, int x, int y);`
  - `output` is engine output (type `Render_Engine`)
  - `context` is engine context
  - `surface` is the dirty region (see below)
- evas calls `output_redraws_rect_add` to add a dirty region to an `Evas`.  When
  rendering, it calls `output_redraws_next_update_get` to return an opaque
  handle that describes an update.  Each object is then rendered with the
  handle.  When all objects have been rendered to the handle,
  `output_redraws_next_update_push` is called.
  - On software x11 engine, the handle has a buffer.  Upon push, the contents is
    copied into an `XImage`, which will later be pushed to the server.

## GL engine

- An image holds a pixel array (e.g. from a png file, system memory etc.).  It
  can also be used to hold a native pixmap through
  `evas_object_image_native_surface_set` (for texture-from-pixmap) on GL engine.
- In GL engine, there are mainly three types of image
  - `evas_gl_common_texture_new`: The pixel array is from user.
  - `evas_gl_common_texture_native_new`: The pixel array is from EGLImage.
  - `evas_gl_common_texture_render_new`: FBO and render-to-texture.
- Whether the pixel array of an image is from user or EGLImage is determined by
  `evas_object_image_native_surface_set`.
- All rendering happens on an opaque surface handle.  Nomally, this opaque
  surface handle represents the window system framebuffer.  For map object, the
  handle represents an FBO.  The rendering will draw to the FBO, and then to the
  winsys fb.
- There are a handful of pritimives.  Their rendering functions are
  - `rectangle_draw`: draw the rectangle
  - `line_draw`: draw the line
  - `polygon_draw`: decompose the polygon into spans and draw the spans
  - `gradient2_linear_draw`: draw to an image and draw the image
  - `gradient2_radial_draw`: draw to an image and draw the image
  - `gradient_draw`: draw to an image and draw the image
  - `image_draw`: draw a textured rectangle
  - `font_draw`: upload the glyph bitmap into an alpha texture, draw the texture
  - `image_map4_draw`: draw a textured rectangle
- Note that all drawing functions
  - must honor the clip, alpha, etc.
  - stores the vertex and calls `glDrawArrays` lazily



struct _Evas_Smart_Class /** a smart object class */
{
   const char *name; /** the string name of the class */
   
   int version;

   /* "add" is object constructor (see evas_object_smart_add) */
   void  (*add)         (Evas_Object *o);

   void  (*del)         (Evas_Object *o);
   void  (*move)        (Evas_Object *o, Evas_Coord x, Evas_Coord y);
   void  (*resize)      (Evas_Object *o, Evas_Coord w, Evas_Coord h); 
   void  (*show)        (Evas_Object *o); // FIXME: DELETE ME
   void  (*hide)        (Evas_Object *o); // FIXME: DELETE ME
   void  (*color_set)   (Evas_Object *o, int r, int g, int b, int a); // FIXME: DELETE ME
   void  (*clip_set)    (Evas_Object *o, Evas_Object *clip); // FIXME: DELETE ME
   void  (*clip_unset)  (Evas_Object *o); // FIXME: DELETE ME

   const void *data;
};
