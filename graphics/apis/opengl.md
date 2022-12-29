OpenGL
======

## History

- for modern versions
  - <https://gitlab.freedesktop.org/mesa/mesa/-/blob/master/docs/features.txt>
- 1.0 was released in 1992
- 1.1 was released in 1997
  - vertex arrays
  - texture objects
- 1.2 was released in 1998
  - 3D textures
- 1.3 was released in 2001
  - multitexturing
  - multitsampling
  - cube maps
  - compressed textures
- 1.4 was released in 2002
  - multi draw arrays
  - depth texture
  - more fixed functions
- 1.5 was released in 2003
  - VBOs
  - occlusion queries
- 2.0 was released in 2004
  - GLSL 1.0 and 1.1
  - MRT
  - npot
- 2.1 was released in 2006
  - GLSL 1.2
  - PBOs
  - sRGB
- 3.0 was released in 2008
  - GLSL 1.3
  - profiles
  - VAOs
  - FBOs
  - conditional rendering
  - floating-point and depth texture formats
  - half-float
  - integer formats
  - texture arrays
  - transform feedback
- 3.1 was released in 2009
  - GLSL 1.4
  - UBOs
  - TBOs
  - instanced rendering
  - buffer textures
  - rectangle textures
- 3.2 was released in 2009
  - GLSL 1.5
  - core and compat profiles
  - geometry shader
  - multisample textures
  - fence sync objects
- 3.3 was released in 2010
  - backport certain 4.0 features
  - GLSL 3.3
  - dual-source blending
  - sampler objects
  - sampler swizzle
  - 1010102
  - instanced arrays
- 4.0 was released in 2010
  - GLSL 4.0
  - tessellation
  - 64-bit
  - indirect draw
  - buffer textures
  - cube map arrays
  - multiple transform feedback
- 4.1 was released in 2010
  - ES2 compat
  - separate shader objects
  - 64-bit vertex attributes
  - viewport array
- 4.2 was released in 2011
  - atomics
  - shader images
  - more formats
- 4.3 was released in 2012
  - ES3 compat
  - compute shader
  - SSBOs
  - array-of-arrays in GLSL
  - multi draw indirect
  - stencil texturing
  - texture views
- 4.4 was released in 2013
  - more immutable storage (`GL_ARB_buffer_storage` and coherent mapping)
  - multi bind
  - bindless texture ext
  - sparse texture ext
- 4.5 was released in 2014
  - ES3.1 compat
  - DSA (direct state access)
  - texture barrier
- 4.6 was released in 2017
  - SPIR-V
  - anisotropic
  - more atomics
  - shader group vote

## OpenGL ES

- 1.0 was released in 2003
  - based on GL 1.3
- 1.1 was released in 2003?
  - based on GL 1.5
- 2.0 was released in 2007
  - based on GL 2.0 without fixed-function
- 3.0 was released in 2012
  - occolusion queries
  - transform feedback
  - instanced rendering
  - MRT >=4
  - ETC2/EAC
  - texturing
    - floating-point textures
    - 3D textures
    - depth textures
    - vertex textures
    - npot textures
    - R/RG textures
    - array textures
    - swizzles
    - seamless cube maps
  - more guaranteed render/texture formats
  - GLSL
- 3.1 was released in 2014
  - compute
  - indirect draws
  - shader images and ssbo
  - texture gather, multisample textures, stencil textures
  - separate shader object
  - GLSL
- 3.2 was released in 2015
  - Android Extension Pack (AEP)
    - ASTC
    - geometry and tessellation shaders
    - per-sample interpolation and shading
    - different blend modes for each RT
    - guaranteed ssbo, images, atomics in FS
  - floating-pint RT
  - TBOs, multisample 2D array, cubemap arrays

## Core Profile

- Deprecation Model
  - some features are marked for deprecation in OpenGL 3.0
  - most of them are removed from OpenGL 3.1 and are added back through `GL_ARB_compatibility`
  - In OpenGL 3.2, removed features are only available in compatibility profile
  - A forward-compatible context removes deprecated features, in addtion to removed features
- Removed Features
  - all objects must be pre-generated with `glGen*`
  - no more color index mode
  - GLSL 1.1 and 1.2
  - glBegin and glEnd
  - edge flags and fixed-function vertex processing
    (`gl*Pointer`, `gl*ClientState`, matrix, frustum/ortho, lighting)
  - client vertex and index arrays (must use buffer object)
  - `glRect*`
  - glRasterPos
  - non-sprite point
  - POLYGON, QUADS, `QUAD_STRIP`
  - PolygonMode and PolygonStipple
  - pixel transfer modes and operations
  - pixel drawing
  - bitmap
  - legacy pixel formats - alpha, luminance, intensity
  - depth texture mode
  - texture CLAMP mode
  - texture border
  - automatic mipmap generation
  - fixed-function fragment processing (TexEnv, Fog)
  - alpha test
  - accumulation buffers
  - glCopyPixels
  - auxiliary buffers
  - evaluators
  - selection and feedback modes
  - display lists
  - glHints
  - attribute stack
  - unified extension string
  - (Deprecated only) glLineWidth

## Chapter 2: OpenGL Fundamentals

- Programmer's View
  - opengl expects hw framebuffer
  - for most part, opengl provides immediate-mode interface
  - calls to open a window into the framebuffer into which the program will draw
  - calls to create a context and associate it with the window
  - some calls draw simple geometric objects
  - some calls affect how the geom. objects are rendered (colored, lit, or mapped)
  - also calls to read/write framebuffer
- Implementator's View
  - divide operations between HW and CPU
  - maintains a large set of states, where some of them are directly controllable by users
- KRH's View
  - a state machine that controls a set of drawing operations
- Fundamentals
  - GL draws primitives subject to a number of selectable modes
  - primitives: point, line segment, polygon, pixel rectangle
  - modes are independent from each other
  - primitives are defined by vertices
  - vertices is associated with data (positional and texture coord, colors, normals)
  - provides parameters such as transformation matrix, lightning, antialiasing, and pixel update ops
  - how GL is initialized, how large is the allocated window are not defined here, but defined by the window system
- States
  - most are server states
  - some are client states
- syntax
  - `rtype Name{1234}{bsifd ub us ui}{v}`
- GL Errors
  - only a subset of errors is detected
  - when error happens, a flag is set and the error code is recorded
  - further errors are ignored, until GetError is called
  - however, there could be multiple flag-code pairs;  multiple GetError clear and return each pairs
- Flushing
  - `glFlush` flushes buffered commands.  It should be called before, for example,
    the program waits for user input.
  - `glFinish` returns only after all previous commands are executed.
  - The buffer being drawn is controlled by `glDrawBuffer`, which is not
    available in OpenGL ES.

## Chapter 3: Dataflow Model

- basic operation

       > display list >
      /                \
    --------------------> evaluator -> per-vertex operation primitive assembly -> rasterization -> per-fragment operations -> framebuffer
    									     /       ^
                                                               pixel operations --- texture memory

## Chapter 4: Event Model

- Asynchronous queries
  - Basic
    - `glGenQueries`
    - `glBeginQuery`
    - `glEndQuery`
    - `glDeleteQueries`
    - It is asynchronous because a begin or command will be added to the command
      queue when a query object begins or ends.  The query does not begin or end
      immediately when the respective function is called.
  - Primitive queries
    - `GL_PRIMITIVES_GENERATED`
    - `GL_TRANSFORM_FEEDBACK_PRIMITIVES_WRITTEN`
  - Occlusion queries
    - `GL_SAMPLES_PASSED`
    - `GL_ANY_SAMPLES_PASSED`
  - Timer queries
    - `GL_TIME_ELAPSED`
    - `glQueryCounter`
    - `glGetIntegerv(GL_TIMESTAMP)`
- `glFenceSync` inserts a fence command in the GL command stream and associates
  it wit the sync object.

## Chapter 5: Shared Objects and Multiple Contexts

## Chapter 6: Buffer Objects

## Chapter 7: Programs and Shaders 

## Chapter 8: Textures and Samplers 

- From the hardware' view, each shader stage have fixed numbers of slots for
  samplers and resource views
  - A resource view is a view of a resource, with swizzling and etc.  It can be
    accessed directly in the shader.  There can be up to hundreds of slots for
    resource views.  
  - A sampler can be used to sample a resource view.  It can be used to access
    a resource view indirectly in the shader, with filtering and etc.  There may
    be only a dozen fo slots for samplers.
- In OpenGL, texture units correspond to samplers.
  - Texture objects are attached to texture units.  Though multiple texture
    objects may be attached to a texture unit (with different targets), only one
    of them can be active.  So there cannot be more textures than there are
    texture units.
  - Resource views are determined from the texture objects.  HW sapmlers are
    determinted by both `glTexParameter` and `glTexEnv` unless there are GL
    sampler objects bound.
- `GL_MAX_VERTEX_TEXTURE_IMAGE_UNITS`, `GL_MAX_GEOMETRY_TEXTURE_IMAGE_UNITS`,
  and `GL_MAX_TEXTURE_IMAGE_UNITS` are the maximum numbers of texture image
  units available to vertex, geometry, and fragment shaders.
  - however, the three shaders combined cannot use more than
    `GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS`
- `glActiveTexture` specifies the active texture unit selector,
  `GL_ACTIVE_TEXTURE`
  - It cannot exceed `GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS`
  - `glTexImage` modifies the texture object bound to the specified target of
    the active texture unit
- In compatibility profole,
  - `glClientActiveTexture` is used to select the client state modified by
    `glTexCoordPointer` and `glEnableClientState`
  - a texture unit is further divided into two sub-units: texture coordinate
    unit and texture image unit
    - A texture unit may or may not have both sub-units.  The numbers of each
      sub-units are given by `GL_MAX_TEXTURE_COORDS` and
     `GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS`.  The former also decides the number
      of sets of texture coordinates of a vertex, the number of texture
      matrices, and texture coordinate generation states.
    - because of this, `GL_ACTIVE_TEXTURE` cannot exceed the larger number of
      `GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS` and `GL_MAX_TEXTURE_COORDS`
    - In core profile, a texture unit is a texture image unit.
  - `GL_MAX_TEXTURE_UNITS` is the number of fixed-function texture units. It is
    usually the number of texture units that have both sub-units.
  - Or to look back from the history,
    - `GL_ARB_multitexture` adds `GL_MAX_TEXTURE_UNITS`, `GL_ACTIVE_TEXTURE`,
      and `GL_ACTIVE_CLIENT_TEXTURE`
    - GL 2.0 adds `GL_MAX_TEXTURE_IMAGE_UNITS` and `GL_MAX_TEXTURE_COORDS`, and
      add the idea that a texture unit has two sub-units
    - core profile drops `GL_MAX_TEXTURE_COORDS` because there are only generic
      attributes
  - see `GL_ARB_fragment_program` to find out what operations use
    `GL_MAX_TEXTURE_COORDS` and what operations use `GL_MAX_TEXTURE_IMAGE_UNITS`

- There are pixel storage operations and pixel transfer operations
  - the former affects things like LSB/MSB, byte swapping, etc.
- Data in client buffer or PBO is said to be packed
  - they go through unpack pixel storage op and then pixel transfer op to reach fb
  - this is the path `DrawPixels` takes
- Data in fb is said to be unpacked
  - they go through pixel transfer op and then pack pixel storage op to reach client
    buffer or PBO
  - this is the path `ReadPixels` takes
- Texture
  - The width, height, and data passed to `glTexImage` include border, which is at
    max 1.
  - the internal dimensions of a texture must be power-of-two
    - `internal width = specified width - 2 * border`
    - `internal height = specified height - 2 * border`
  - Each texel in the image array is assigned an coordinate `(i, j, k)`
    - `i` is in `[-b, specified width - 1 - b]`
    - `j` is in `[-b, specified height - 1 - b]`
    - `k` is in `[-b, specified depth - 1 - b]`
  - wrapping and texcoord `(s, t, r, q)` does not include border
    - border is used by linear sampling
  - nearest sampling
    - given `(s, t, r)`
    - wrap mode is applied first on it
    - let `(u, v, w)` be `(s * 2^n, t * 2^m, r * 2^l)` 
    - `i` is `[u]` if `s < 1`; or `i` is `2^n - 1` if `s = 1`
    - no border is sampled
  - linear sampling
    - `i0` is `[u - 1/2]`. `j0` and `k0` are similar.
    - the texture value is the averaging of the 2, `2*2`, or `2*2*2` texels, starting
      at `(i0, j0, k0)`
    - border may be sampled
    - if the texel is even beyond the border, constant border color is used
      - `GL_TEXTURE_BORDER_COLOR`
  - In mesa, `struct gl_texture_image` treats the image data an normal image array
    - border is added to `(i, j, k)` to form non-negative coordinates
    - see `sample_1d_nearest` and `sample_1d_linear`
  - For hardware, the buffer for the texture has a miptree layout
    - there may be one or more levels (mipmapping)
    - there may be one or more layers (arbitrary for texture arrays, 6 for cube
      map, 1 otherwise)
    - depth is always 1, unless 3D
    - then width and height
    - the driver usually allocates a buffer of size which is the sum of
      `width * height * cpp * (layer || depth)` for each level
    - OpenGL cannot render to more than one layer at a time
- `glTexImage*D` decodes a texture image and stores it in a texel array
  - The buffer is called a texture image
  - the process is called decoding
  - the decoded image is called a (3-dim) texel array
  - a texel is derived by indexing a texel array using `(i, j, k)`
- Groups
  - there are integer or floating-point color component groups
  - floating-point depth component groups
  - stencil index groups
  - floating-point depth component and stencil index groups
- Pixel Transfer
  - user data undergo pixel transfer first
  - the type decides the elements/indices of the buffer
    - it may be signed or unsigned byte, short, int; half or full float; or
      packed
  - the format is used to assign meanings to and group the elements/indices
    - `STENCIL_INDEX`: a group has one element, meaning stencil index
    - `DEPTH_COMPONENT`: a group has one element, meaning depth component
    - `DEPTH_STENCIL`: a group has two elements, meaning depth and stencil index
    - `RED`, `GREEN`, `BLUE`: a group has one element, meaning the respective
      color components
    - `RG`: a group has two elements, meaning red and green components
    - `RGB`: a group has three elements, meaning red, green, and blue components
    - `RGBA`: a group has four elements, meaning red, green, blue, and alpha
      components
    - `BGR`, `BGRA`: same as `RGB` and `RGBA`, with reversed order
    - `<color-format>_INTEGER`: same as the respective color format, except that
      the components are integer instead of float
      - that is, skip the "convert to float" step
  - groups of floating-point components are converted to floating-point
    - that is, all format except `<color-format>_INTEGER` and stencil index
    - unsigned integer is converted to `[0, 1]`
    - signed integer is converted to `[-1, 1]`
  - finally, for color component groups, each group is expanded to RGBA.
- Internal texture format
  - the transfered groups are processed and stored in internal texture format
  - first, all components are clamped; indices are masked, according to the
    internal format
  - then all components are selected (discarding unwanted components) according
    to the internal format
  - the valid base formats are `DEPTH_COMPONENT`, `DEPTH_STENCIL`, `RED`, `RG`,
    `RGB`, and `RGBA`
- A texel is formed by selecting the components of a group from pixel transfer
  using the internal format
  - it is assigned a integer coordinate `(i, j, k)`
- A component of a texel may be
  - unsigned normalized fixed-point
  - signed normalized fixed-point (SNORM)
  - unsigned integer (UI)
  - signed integer (I)
  - float (F)
  - all types except the first one and float are undefined for fixed-function
    fragment processing
- Depth Textures
  - A texutre of internal format `GL_DEPTH_COMPONENT` or `GL_DEPTH_STENCIL` is
    called a depth texture
  - A depth texture is treated as a `GL_LUMINANCE` texture initially.  It can be
    modified by modifying `GL_DEPTH_TEXTURE_MODE`.
    - it is always treated as a `GL_RED` texture in GL 3.2 core profile.
  - stencil values are ignored
  - The texel value `D_t` is used to give
    - `r = D_t`, when `GL_TEXTURE_COMPARE_MODE` is `GL_NONE`
    - `r = 1.0 or 0.0` depending on `GL_TEXTURE_COMPARE_FUNC`, when
      `GL_TEXTURE_COMPARE_MODE` is `GL_COMPARE_REF_TO_TEXTURE`
    - r is the final value.  Its interpretation depends on
      `GL_DEPTH_TEXTURE_MODE`.

## Chapter 9: Framebuffers and Framebuffer Objects

## Chapter 10: Vertex Specification and Drawing Commands

- The last paragraph of Vertex Specification says
  - 4 floats for each texture unit
  - 3 floats for normal
  - 1 float for fog
  - 4 floats for color
  - 4 floats for secondary color
  - 1 float for color index
  - no vertex
- A vertex can have at most `GL_MAX_VERTEX_ATTRIBS` generic vertex attributes
- In compatibility profile, it can have additionally
  - vertex coordinates
  - normal
  - color
  - secondary color
  - color index
  - edge flag
  - fog coordinates
  - and `GL_MAX_TEXTURE_COORDS` sets of texture coordinates
- Compatibility arrays are manipulated by
  - `glEnableClientState` and `glDisableClientState` to enable/disable an array
  - `glXxxPointer` to specify the array
  - There are multiple sets of texture coordinates and the one to be
    enabled/disabled or specified is determinated by `GL_CLIENT_ACTIVE_TEXTURE`.
    It can be set `glClientActiveTexture`.
- In contrast, generic vertex attributes are maniuplated by
  - `glEnableVertexAttribArray` and `glDisableVertexAttribArray`
  - `glVertexAttribPointer` and `glVertexAttribIPointer`
- `DrawArrays` is equivalent to calling `ArrayElement` with a loop
  - everything is in floats.
- Everything is stored in floats.  Conversions happen for normal and colors.
  - the conversion is described in final color processing.
- vertices, normal, and texcoords are transformed.
- The output of colors to rasterization are fixed-point, unless fragment shader
  is enabled.
- Effective Drawing

    A mesh is described by a vertex buffer, an index buffer, and many primitive
    
      struct primitive {
        GLenum mode;
    
        GLsizei num_vertices;
        GLsizei offset;
      };
    
      for each prim:
          glDrawElements(prim->mode, prim->num_vertices, GL_UNSIGNED_SHORT, prim->offset);
    
    The number of vertices is fixed.  For effective drawing, we can only reduce
    the number of indices and primitives.

    Now some primitives may share the same shape, but with differen position or
    size.  Two such primitives differ in the vertices, and their indices have a
    constant displacement.  Thus, we can have
    
      struct primitive {
        GLenum mode;
    
        GLsizei num_vertices;
        GLsizei offset;
        GLint basevertex;
      };
    
      for each prim:
          glDrawElementsBaseVertex(prim->mode, prim->num_vertices, GL_UNSIGNED_SHORT, prim->offset, prim->basevertex);
    
    with two such primitives share the same "offset" but different "basevertex".
    
    More often than usual, the mode of the primitives does not change that much.
    It makes sense to merge pritimives to save the calling overhead
    
      struct primitive {
        GLenum mode;
        GLsizei prim_count;
    
        GLsizei *num_vertices;
        GLsizei *offsets;
        GLint *basevertices;
      };
    
      for each prim:
          glMutlDrawElementsBaseVertex(prim->mode, prim->num_vertices, GL_UNSIGNED_SHORT, prim->offsets, prim->prim_count, prim->basevertex);
    
    However, the saving of the calling overhead is not big enough (Only in
    st_draw_vbo, not in the driver).  There is also an additional cost now that
    some fields of a primitive are dynamically allocated.  Another way to
    effectively merge primitives is to insert a restart/cut index.
- Conditional rendering
  - `glBeginConditionalRender`
  - `glEndConditionalRender`
  - It is used with the result of an occlusion query.  An example is to draw the
    bounding box of a complex object with occlusion query and then draw the
    object with conditional rendering.

## Chapter 11: Programmable Vertex Processing

## Chapter 13: Fixed-Function Vertex Post-Processing

- Transform Feedback
  - `glBeginTransformFeedback`
  - `glEndTransformFeedback`
  - `glTransformFeedbackVaryings`

## Chapter 14: Fixed-Function Primitive Assembly and Rasterization

- Rasterization
  - a primitive is rasterized using solely the positions of the vertices.  Each
    pixel occupied by the primitive is passed to the fragment shader.
    - when `GL_SAMPLE_BUFFERS` is zero, single-sample rasterization is used
    - when `GL_SAMPLE_BUFFERS` is one, multisample rasterization is used.  The
      algorithm of multisample rasterization depends on `GL_MULTISAMPLE`.
    - `GL_MULTISAMPLE` is not effective to single-sample rasterization.
  - A pixel occupied by a primitive, along with its depth value and varying
    vertex shader outputs, is called a fragment.  The varying vertex shader
    outputs are called the the associated data of the fragment.
    - The associated data are derived from interpolating the associated data of
      the vertices of the primitive.  There are several interpolation methods
      defined.  The depth value is also interpolated.  But there is only one
      interpolation method.
  - Fragments are routed through fragment shader before going through
    per-fragment operations and hitting the framebuffer.
- Multisample Rasterization
  - each fragment has a coverage value of `GL_SAMPLES` bits, where single-sample
    rasterization does not.
  - each fragment includes `GL_SAMPLES` depth values and `GL_SAMPLES` sets of
    associated data, where single-sample rasterization includes a single depth
    value and a single set of associated data.
  - Color, depth, and stencil values of a sample are stored in the multisample
    buffer.  There will be no depth and/or stencil buffer.  Color buffers do
    coexist with the multisample buffer, however.
  - If `GL_MULTISAMPLE` is disabled, multisample rasterization of all primitives
    is equivalent to single-sample (fragment-center) rasterization, except that
    the fragment coverage value is set to full coverage. The color and depth
    values and the sets of texture coordinates may all be set to the values that
    would have been assigned by single-sample rasterization, or they may be
    assigned as described below for multisample rasterization.
    - That is, as if `GL_SAMPLES` sample points are all at the center of the
      pixel?
  - multisample rasterization differs significantly from single-sample
    rasterization when `GL_MULTISAMPLE` is enabled.
- Rasterization of POINTs
  - a pixel is occupied by a POINT if the pixel lies in a square centered at the
    POINT.  The associated data corresponds to those of the vertex of the POINT.
    - For LINEs or TRIANGLEs, the depth value and the associated data of a pixel
      are interpolated using the vertices of the primitive.  The depth value
      must be noperspective evaluated.
  - for multisample rasterization with `GL_MULTISAMPLE` on, a pixel has
    `GL_SAMPLES` sample points.  A pixel is occupied by by a POINT if any of the
    sample points lies in a sqaure centered at the POINT.  A bit of the coverage
    value is set if and only if the corresponding sample point lies in the
    square.  The associated data of each sample point corresponds to those of the
    vertex of the POINT.
    - For LINEs or TRIANGLEs, the depth value and the associated data of a
      sample point are interpolated using the vertices of the primitive.  The
      depth value must be noperspective evaluated at the location of the sample
      point.  The associated data may be perspective evaluated at any location
      (that is, not necessarily the sample point) within the pixel.  Any two
      associated data values (say, color and texcoord) need not be evaluated at
      the same location.
- Fragment Shader
  - With multisample rasterization, an implementation is allowed to assign the
    same assocaited data to multiple or all sample points.  In that case, the
    implementation is safe to run the fragment shader once per fragment when the
    shader does not use the depth value.
  - Otherwise, the fragment shader must be run per sample point.
- Per-Fragment Operations
  - For multisample rasterization with `GL_MULTISAMPLE` enabled, Per-fragment
    operations operate on sample points (whose coverage bits are set) of a
    fragment rather than the framgment.  Failure of the stencil or depth test
    results in discarding the sample point rather than the fragment.  All
    operations are performed on the multisample buffer (rather than the color
    buffers or the non-existing depth/stencil buffer).
  - For multisample rasterization with `GL_MULTISAMPLE` disabled, same as above
    except that a certain optimization may be performed.
  - Finally, the color values in multisample buffer are resolved to the color
    buffers as specified by `glDrawBuffers`.
- Whole Framebuffer Operations
  - `glColorMask` and `glDepthMask` and others control the mask for the
    multisample buffer.
  - `glClear` clears the multisample buffer
  - `glReadPixels` with `GL_DEPTH_COMPONENT` reads the depth values in the
    multisample buffer.  The implementation decides how to resolve the depth
    values of a pixel.  Same to `GL_DEPTH_STENCIL` and `GL_STENCIL_INDEX`.
    - But it fails when reading a multisample FBO.
  - `glBlitFramebuffer` resolves if needed.
- Multisampling
  - `GL_MULTISAMPLE` multisample rasterization (boolean)
  - `GL_SAMPLE_ALPHA_TO_COVERAGE` modify coverage from alpha (boolean)
  - `GL_SAMPLE_ALPHA_TO_ONE` set alpha to maximum (boolean)
  - `GL_SAMPLE_COVERAGE` mask to modify coverage (boolean)
  - `GL_SAMPLE_COVERAGE_VALUE` coverage mask value
  - `GL_SAMPLE_COVERAGE_INVERT` invert coverage mask value (boolean)
  - `GL_SAMPLE_MASK` sample mask enable (boolean)
  - `GL_SAMPLE_MASK_VALUE` sample mask words
- Textures (state per texture image)
  - `GL_TEXTURE_SAMPLES` number of samples per texel
  - `GL_TEXTURE_FIXED_SAMPLE_LOCATIONS` whether the image uses a fixed sample
    pattern
- Renderbuffers (state per renderbuffer object)
  - `GL_RENDERBUFFER_SAMPLES` number of samples
- Implementation dependent values
  - `GL_MAX_INTEGER_SAMPLES` maximum number of samples in integer format multisample
    buffers
  - `GL_MAX_COLOR_TEXTURE_SAMPLES` maximum number of samples in a color multisample
    texture
  - `GL_MAX_DEPTH_TEXTURE_SAMPLES` maximum number of samples in a depth/stencil
    multisample texture
  - `GL_MAX_SAMPLE_MASK_WORDS`
- Framebuffer dependent values
  - `GL_SAMPLES` coverage mask size
    - For FBO, it is equal to `GL_TEXTURE_SAMPLES`/`GL_RENDERBUFFER_SAMPLES` of
      the attached renderbuffers/textures.
  - `GL_SAMPLE_BUFFERS` number of multisample buffers
    - For FBO, if `GL_SAMPLES > 0`, its value is set to 1.
  - `GL_SAMPLE_POSITION` explicit sample positions
  - `GL_MAX_SAMPLES` maximum number of samples supported for multisampling
    - not a impl dependent value?

## Chapter 15: Programmable Fragment Processing

## Chapter 17: Writing Fragments and Samples to the Framebuffer

- Stencil and depth tests happen at fragment operation.
- The depth buffer is used for depth test.  It is enabled by
  `glEnable(GL_DEPTH_TEST)`.
- For objects that have been drawn, their depth information is stored in the
  depth buffer.  When a new object is to be drawn, only pixels with z satisfying
  `glDepthFunc()` are drawn.
- Stencil are used for multi-pass rendering.  The first pass draws some object
  to the stencil buffer to create a mask.  The second pass draws another object
  to the color buffer with the stencil buffer as the mask.
- e.g. a ball is reflected by the floor.  We draw the floor to the stencil
  buffer.  And then draw the reflection of the ball with the stencil buffer.
- How to draw a scene with translucent objecsts
  1. Draw opaque objects (the scene) first
  2. Disable depth test
  3. Draw translucent objects, from farthest to nearest
- Lesson 09 of NeHe shows a cheap way to blend an irregular RGB (no alpha)
  bitmap onto the scene.  It uses `glBlendFunc(GL_SRC_ALPHA, GL_ONE)`, with the
  unwanted part of the bitmap being black.

## Chapter 18: Reading and Copying Pixels

- `glReadPixels` reads data from the frame buffer.

## Chapter 19: Compute Shaders

## Chapter 20: Debug Output

## Chapter 22:  Context State Queries

## Legacy Concepts

- Begin/End
  - vertex are input and transformed, asscociated with data to form a processed vertex
  - the coordinates (vetex's and current normal's) are transformed
  - associated data relate to current color, edge flags, texture coord sets, etc.
- Vertex
  - Vertex data enter `Vertex Operation` first.  It is tranformed by
    `GL_MODELVIEW` from object coordinates to eye coordinates.  Following that,
    they enter `Primitive Assembly` where they are transformed again by project
    matrix then clipped by viewing volume clipping planes, from eye coordinates to
    clip coordinates.  After that, perspective division by w occurs and viewport
    transform is applied in order to map 3D scene to window space coordinates.
  - Rasterization turns geometric and pixel data into fragments.  Each fragment
    corresponds to a pixel in the frame buffer.
  - In the final stage is `Fragment Operation`.  A texture element is generated
    for each fragment and is applied.  Fog calculations are applied.  After that,
    tests like `Scissor Test`, `Alpha Test`, `Stencil Test`, and `Depth Test` are
    done in order.  Finally, blending, dithering, etc. are performed, and actual
    pixel data are stored in the frame buffer.
- Transformation
  - Coordinates changes:
      [Vertex Data] -> Object Coord. -> [ModelView] -> Eye Coord. -> [Projection]
      -> Clip Coord. -> [Divide by w] -> Normalized Device Coord. -> [Viewport] ->
      Window Coord.
  - There is only a `MODELVIEW` matrix.  However, we can think of it as consisted
    by `MODEL` matrix and `VIEW` matrix.
  - `MODELVIEW = VIEW * MODEL`.  Vectors are multiplied by `MODEL` matrix to
    become world coord. and then multiplied by `VIEW` matrix to become eye coord.
  - When drawing, issue viewing commands first.  And then modeling commands.
  - That is because of the order of matrix multiplication.  Say, current matrix is
    `C` and the new matrix is `V`.  `glMultMatrix` will result in `C * V`.
  - Following this convention, viewing commands are read in the code order;  while
    modeling commands should be read in the reversed order.
- Normal Transformation
  - normal and vector are perpendicular, i.e. `dot(n,v)` is 0.  dot(Xn,Mv) should
    also be 0, where M is the model-view; X is the normal transformation.  Some
    calculations show that X is the transpose of the inverse of M.
- Viewing Volume
  - `glFrustum` and `glOrtho` are, in some sense, the same as `glTranslate` or
    `glRotate`.  They all multiply current matrix.  Thus, remember to call
    `glMatrixMode` to use `GL_PROJECTION` first.
  - Talking about projection, one should note that the camera is at `(0, 0)`
    looking `-z`.  Forget the virtual `VIEW` matrix.
  - The vertices are clipped by near and far planes.  The size is proportional to
    the plane size.  That is, if the far plane is twice the size of near plane,
    an object moving from near plane to far plane will appear half the size.
- Pixel Transfer
  - `CopyPixels` is the combination of `ReadPixels` and `DrawPixels`
    - the data go through pixel transfer op of `ReadPixels` and are written
      directly to fb.
  - `DrawPixels`
    - Groups and elements.  There are four kinds of pixel groups.  RGBA
      components, Depth component, Color index, and Stencil index.
    - Unpacking.  Usually, the whole client buffer (or PBO) is used.  It is
      possible to select a subimage by giving `UNPACK_ROW_LENGTH`,
      `UNPACK_SKIP_PIXELS`, and `UNPACK_SKIP_ROWS`.
    - Conversion to float-point.  If the format gives groups of components, each
      component is converted to float point.
    - Conversion to RGB.  If the format gives luminance, it is expanded into RGB.
    - Conversion to RGBA.  If a group has only RGB, 1.0 is given for A.
    - Pixel Transfer Op.  It works on groups.
    - Final Conversion.  Clamping to `[0, 1]`.
    - Conversion to Fragments.  This is where `PixelZoom` comes into play.
  - Data in `TexImage[123]D` are processed just like `DrawPixels`.  They stop just
    before final conversion, and are clamped to `[0, 1]`.
