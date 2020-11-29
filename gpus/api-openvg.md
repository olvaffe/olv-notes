OpenVG
======

## Pipeline

* OpenVG supports single-sampled or multi-sampled surfaces
* Stages
  * Path, Transformation, Stroke, and Paint
  * Stroked Path Generation
  * Transformation
  * Rasterization
  * Clipping and Masking
  * Paint Generation
  * Image Interpolation
  * Color Transform, Blending, AA
* Path, Transformation, Stroke, and Paint
  * The app defines the path, transform, stroke, and paint.
  * `vgDrawPath` is used to initiates the render process
    * it may be fill or stroke or both.  When both, it fills first and then
      strokes.
  * Another way to initiate the render process is `vgDrawImage`.
    * the current path is set to a rectangle bounding the image.
* Stroked Path Generation
  * If the path is to be stroked, the path is converted to a new one by
    the current stroke parameters.  The rest of the pipeline sees only the
    converted path.
* Transformation
  * path-user-to-surface transforms a path
  * image-user-to-surface transforms an image
* Rasterization
  * Every pixel affected by the path is assigned a coverage value and the value
    is saved for later use.
  * For single-sampled surface, the sample point coincides with pixel center.
  * ...
* Clipping and Masking
  * Pixels not lying in the drawing surface and, if scissoring is enabled,
    within the union of the scissor rectangles are assigned a coverage value of
    0.
  * An app specified mask image is used to modify the computed coverage values.
    Each coverage value is multiplied by the mask value for the corresponding
    pixel.
  * If the resulting coverage value of a pixel is zero, it is skipped in the
    remainder of the pipeline.
* Paint Generation
  * At each pixel, the current fill or stroke paint is used to define a color
    and an alpha value.
  * For gradient and pattern paints, a paint transformaion generated from
    paint-to-user and path-user-to-surface transformations.  It transforms the
    paint.
  * This stage may be skipped for operations that do not require a paint.
  * For multi-sampled surfaces, paint genration may happen per sample point or
    per pixel.
* Image Interpolation
  * If image drawing is not taking place, this stage is skipped.
  * An image color and alpha value is defined at each pixel by interpolating the
    image values using the inverse of image-user-to-surface transformation.
  * The image color/alpha and the paint color/alpha are combined according to
    the current image drawing mode.
* Color Transform, Blending, AA
  * At each pixel, the color/alpha values from previous stage are passed through
    a color transformation and converted into the destination colorspace.
  * The resulting colors are blended with the destination color and alpha values
    according to the current blending rule.
  * The coverage value of the pixel is used to interpolated between the original
    destination color and the new blended result to produce an antialiased
    result.

## Rasterization

* See `Rasterizer::fill` and `pixelPipe::pixelPipe` methods from the reference
  implementation
* MSAA
  * For each pixel, there can be multiple sample points
    * a sample mask is used to record which sample points are inside the path
  * A coverage value for the pixel is determined using the sample mask
    * a weighted filter may be used
    * it is 1.0 when the sample mask is full; 0.0 when the sample mask
      is empty
  * the pixel, along with its sample mask and coverage value, is passed down to
    the pipeline
    * for MSAA, if the sample mask is empty, the pixel is discarded
    * for normal AA, if the coverage is 0.0, the pixel is discarded
    * otherwise, the paint color/alpha for `(x+0.5, y+0.5)` is generated
    * if imaging is enabled, the image color/alpha for `(x+0.5, y+0.5)` is also
      generated
    * the pixel color/alpha is derived from the paint color/alpha and image
      color/alpha, depending on the image mode
    * color transformation is also applied
  * finally, the pixel color/alpha is blended with the surface color/alpha to
    produce the final value.  The final value is then multiplied by the coverage
    value and the alpha mask value
    * or in the MSAA case, the alpha mask may store the sample masks instead of
      coverage values.  In other words, the alpha mask stores per-sample
      coverage values of 1 bit
* `vgDrawImage` is conceptually drawing a rectangular path, except
  * image-user-to-surface is used instead of path-user-to-surface
  * imaging enabled
* `vgRenderToMask` is conceptually drawing a path, except
  * the surface is not written to
  * the mask is updated

## Colors

* surface is in non-premultiplied, sRGBA format
* images may be in a lot of formats
* conversions between color spaces (lRGB, sRGB, lL and sL) are well-defined
  * alpha is always linear
* color interpolation takes place in premultiplied form
* format conversion
  * first, premultiplied to non-premultiplied
    * colors are clamp to `[0, alpha]` first
    * if alpha is non-zero, colors are dividied by it
  * if the colorspace differs,
    * each chanel is normalized to `[0, 1]`
    * color space conversion
    * each channel is scaled back using the dst format
  * if the format is the same but the depths differ
    * simple scale and rounding
* Given a surface pixel `(x, y)` and its coverage `coverage`, how is the color
  assigned?  In RI,
  * convert paint color to premultiplied
  * read image color, premultiplied.  Convert paint color to image color space.
  * convert to non-premultiplied for color transformatin and convert back
  * convert to destination color space (still premultiplied)
  * read the surface pixel for blending, premultiplied first
  * convert to surface format before writing
* For Vega, we might want to
  * add a stage between paint and image for format conversion
    * when `VG_DRAW_IMAGE_MULTIPLY`, convert to image colorspace
    * when `VG_DRAW_IMAGE_STENCIL`, convert to paint colorspace
    * always premultiplied
  * add a stage between image and color transformation
    * color transformation expects unpremultiplied colors
  * add a stage between color transformation and blend
    * blending is in destination colorspace
    * it is premultiplied
  * move alpha mask after blend
    * do coverage in linear space

## Objects

* fonts, images, mask layers, paints, paths
* reference counted
* `VG_INVALID_HANDLE` is 0
* Handles may only be shared with the shared contexts
* An image is in use by OpenVG when it is
  * a parent or child of other images,
  * set as a paint pattern, or
  * set as a glyph image.
* An image is in use by EGL when it is set as a render target
  * `eglCreatePbufferFromClientBuffer`
* An image may not be in use in both OpenVG and EGL

## Misc

* all pointers must be aligned to the size of the data type
* `vgGetError` gets the older error since the last call and clears the error

## Parameters

* `vgSet` and `vgGet` are relative to the current context
* `vgSetParameter` and `vgGetParameter` are relative to the specified handle
* for intergral parameters, `vgSetf` is equivalent to `vgSeti` with the value
  floored, and clamped to the nearest valid integer
  * Likewise for `vgGeti` on float parameters
* `vgSet` on read-only parameters is no-op (not an error)

## Rendering Quality

* it may be non-anti-aliased, faster, or better
* it has not effect on multisampled surfaces
* `VG_PIXEL_LAYOUT` may be set to enable subpixel rendering

## Coordinate Systems

* User coordinate system and surface coordinate system have their origins at the
  lower-left pixel
* They both differ from the coordinate system of the window system usually
* Homogeneous coordinates, `[x, y, 1]`, are used to allow projective
  transformations
* There are 2 paint-to-user matrices for fill and stroke paints
* There are 3 user-to-surface matrices for path, image, and glyph
* Only the image-user-to-surface matrix can be projective

## Masking

* there is a drawing surface mask that defines an additional coverage value at
  each sample of the drawing surface
* the value modifies the coverage value computed by the rasterization stage
* Masking is enabled when `EGL_ALPHA_MASK_SIZE > 0` and `VG_MASKING` is true
* Masking must be supporeted by an implementation

## Clearing

* `VG_CLEAR_COLOR` is non-premultiplied sRGBA
* Clipping and scissoring apply, but antialiasing, masking, and blending do not
  occur

## Paints

* The paint color and alpha of a pixel at surface coordinates `(x, y)` is
  defined by a sample point in the paint coordinate space
  * `(x, y) := user-to-surface * paint-to-user * sample-point`
* Color paints
  * the color of the sample point is a constant defined by `VG_PAINT_COLOR` is
    non-premultiplied sRGBA
  * `vgSetColor` is a short-hand to modify the color
* Gradient paints
  * a float value of a sample point is derived from the gradient function.  The
    value is used to look up the color in the color ramp.
  * a linear gradient is a function `g(x, y)` defined by two endpoints.
    * for each sample point in the pait coordinate space, a float value can be
      derived using the function
  * a radical gradient is a different function defined by a center, a focal
    point, and a radius
  * color ramp
    * a color ramp has many stops with offsets between `[0, 1]`
    * the float value from the gradient function is used to look up the color
      from the ramp
    * `VG_COLOR_RAMP_SPREAD_MODE` specifies how the color is mapped if the float
      value is outside of `[0, 1]`
* Pattern paints
  * the color of the sample point is looked up in the `VGImage`.  Each pixel
  `(x, y)` of the image defines a point of color at the pixel center
   `(x + 0.5, y + 0.5)`
  * `vgPaintPattern`
  * `VG_PAINT_PATTERN_TILING_MODE`

## Reference Implementation

* Support `EGL_VG_COLORSPACE` and `EGL_VG_ALPHA_FORMAT`
* An `EGLSurface` is internally a `Drawable`
  * has a color `Surface` and a mask `Surface`
* A `Surface` has an `Image` for storage

## Vega

* Mesa's implementation of OpenVG 1.0
* A `vg_context` has a member `state`.  It remembers the states of g3d and
  OpenVG
* `vg_validate_state`
  * propogate OpenVG states to g3d states
  * not all OpenVG states are g3d states
* `vg_manager_validate_framebuffer`
  * A current context always has a draw buffer (`st_framebuffer`)
    * there is a color rb
    * there is a depth/stencil rb
      * depth is used for scissoring; `VG_SCISSOR_RECTS` region has depth 0.0;
        the rest has depth 1.0;  the depth function is `PIPE_FUNC_GEQUAL`
      * stencil is used for path silhouette;  a point outside a path has stencil
        value 0; a point inside a path has stencil value != 0; rendering has two
        passes; the first pass updates only the stencil; the second pass draws
        the path and reset the stencil
    * there is an alpha mask sampler view
      * for each pixel, its coverage value is multiplied by the alpha of the
        texture.
    * there is a blend sampler view
      * when an advanced blending mode is used, the current draw buffer is
        copied into the blend sampler view for sampling
      * or for advanced mask operations, the current alpha mask is copied into
        the blend sampler view for sampling
  * the draw buffer needs to be validated
  * the color texture from the system window is used as the color rb
  * a depth texture is created for depth rb
  * `pipe_framebuffer_state` is updated
  * `VIEWPORT_DIRTY` and `DEPTH_STENCIL_DIRTY` is set
  * `setup_new_alpha_mask` creates the alpha mask sampler view and initializes
    it (to all one's)
* Renderer
  * `vgDrawPath` calls `renderer_draw_quad`
  * `vgDrawImage` calls `renderer_texture_quad`
  * `vgMask` calls `renderer_draw_texture`, which is almost equivalent to
    `renderer_texture_quad`
  * `vgCopyImage` calls `renderer_copy_texture`
  * `vgCopyPixels`, `vegaSetPixels`, `vgGetPixels` call `renderer_copy_surface`,
    which is similar to `renderer_copy_texture`, but it also deals with source
    equal to destination.
* Shaders
  * normally, the vertex shader is `vs_plain_asm`.
    * When `vgClear` is called, VS is switched to `vs_clear_asm` temporarily and
      restored.
    * When the renderer is used to copy between resources/surfaces or render
      with texture, VS is switched to `vs_texture_asm` temporarily and restored.
  * normally, the fragment shader is constructed by `shaders_cache_fill`.
    * When `vgClear` is called, FS is switched to
      `util_make_fragment_passthrough_shader` temporarily.
    * When scissoring is enabled, FS is switched to ` pass_through_depth_asm`
      temporarily to update the depth buffer.
    * When `vgMask` is called, FS is switched to one of `solid_fill`,
      `set_mask_asm`, `union_mask_asm`, `intersect_mask_asm`, or
      `subtract_mask_asm` temporarily
* Constants
  * the VS constants are always two vectors to scale and translate the
    position in OpenVG surface coordinates to `[-1, 1]`, the clipped coordinates
  * normally, the FS constants are 8 or 20 floats depending on the paint type.
    For solid color, the first 4 floats are the paint color.  For others, they
    may describe the parameters of a gradient
    * When `vgMask` is called, the constans are 4 floats for the color of the
      mask fill (alpha 0 or alpha 1) or 4 ones when the mask operation is not
      fill or clear
* Samplers
  * normally,
    * Sampler 0 is used for gradient or pattern paint.  Normalized.  The sampler
      view is that of the paint.
    * Sampler 1 is used for masking.  Non-normalized.  The sampler view is that
      of the mask.
    * Sampler 2 is used for advanced blending.  Non-normalized.  The sampler
      view is the blend sampler view.  `vg_prepare_blend_surface` copies the
      contents of the draw buffer to the sampler view just in time.
    * Sampler 3 is used for drawing images.  Normalized.  The sampler view is
      that of the image.
* Vertex Shaders
  * all `vs_plain_asm`, `vs_clear_asm`, and `vs_texture_asm` expect
    * input 0 to be the positions
    * two constant vectors to scale and translate the positions
  * `vs_clear_asm` expects input 1 to be the clear color.  It is MOVed to output
    directly.
  * `vs_texture_asm` expects input 1 to be the tex coords.  It is MOVed to
    output directly.
* Objects
  * `vg_paint`
  * `vg_image`
  * `vg_mask`
  * `vg_font`
  * `vg_path`
* `vg_paint`
  * solid, gradient, pattern
* `struct path`
  * `segments`
  * `control_points`
