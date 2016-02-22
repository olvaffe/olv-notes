OpenGL 1.0
OpenGL 1.1
OpenGL 1.2
OpenGL 1.3
OpenGL 1.4
OpenGL 1.5
GL_ARB_shading_language_100 -> GLSL 1.0
OpenGL 2.0 -> GLSL 1.1
OpenGL 2.1 -> GLSL 1.2

OpenGL 3.0 (2008)
 - GLSL 1.3
 - deprecation model
 - conditional rendering
 - floating-point color and depth formats
 - FBO
 - texture array
 - transform feedback
 - VAO

OpenGL 3.1 (2009)
 - GLSL 1.4
 - GL_ARB_compatibility
 - instanced draw
 - copy between buffer objects
 - primitive restart
 - texture buffer object
 - uniform buffer object
 - rectangular textures
 - signed normalized texture formats

OpenGL 3.2 (2009)
 - GLSL 1.5
 - Core and Compatibility profiles (glXCreateContextAttribsARB), superceding GL_ARB_compatibility
 - draw elements base vertex
 - multisample texture
 - geometry shaders
 - fence sync objects

OpenGL 3.3 (2010)
 - GLSL 3.3
 - sampler objects

OpenGL 4.0 (2010)
 - GLSL 4.0
 - indirect draw
 - tesselation shader

OpenGL 4.1 (2010)
 - GLSL 4.1
 - ES2 compatibility
 - viewport array


Deprecation Model
 - some features are marked for deprecation in OpenGL 3.0
 - most of them are removed from OpenGL 3.1 and are added back through GL_ARB_compatibility
 - In OpenGL 3.2, removed features are only available in compatibility profile
 - A forward-compatible context removes deprecated features, in addtion to removed features

Removed Features
 - all objects must be pre-generated with glGen*
 - no more color index mode
 - GLSL 1.1 and 1.2
 - glBegin and glEnd
 - edge flags and fixed-function vertex processing
   (gl*Pointer, gl*ClientState, matrix, frustum/ortho, lighting)
 - client vertex and index arrays (must use buffer object)
 - glRect*
 - glRasterPos
 - non-sprite point
 - POLYGON, QUADS, QUAD_STRIP
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
