apitrace
========

## Build

- build
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Release`
  - add `-DENABLE_GUI=OFF` and `-DCMAKE_TOOLCHAIN_FILE=cross.cmake` for
    cross-compile
- optional dependencies
  - `libbrotli` for brotli compression
  - `libprocps` for memory profiling
  - `libdwarf` for backtrace (of `glDrawArrays`, etc)
- distribution
  - `apitrace trace` finds the wrapper library in
    - `./wrappers`
    - `../lib/apitrace/wrappers`
    - `$libdir/apitrace/wrappers`
  - `apitracedir=apitrace-$(date +%Y%m%d)`
  - `ln -sf out $apitracedir`
  - `tar zcf $apitracedir.tar.gz $apitracedir/{apitrace,glretrace,eglretrace,wrappers/*.so}`

## Usage

- TBD

## Tracer

- The tracer is a library to be `LD_PRELOAD`ed
- It overrides all EGL/GLX/WGL/AGL and GL/GLES functions
- It also overrides `dlopen` so that the handle to the tracer is returned when
  the app `dlopen` libEGL/libGL
- For most functions, recording is easy
  - `beginEnter(func name, arg names)`
  - `beginArg(N); write*; endArg()` for each input arg
  - `endEnter()`
  - optionally unwrap the input args
  - do the real function call
  - `beginLeave()`
  - `beginArg(N); write*; endArg()` for each output arg
  - `beginReturn(); write*; endReturn()`
  - optionally wrap the output args and ret
  - `endLeave()`
- `write*` is not too complex
  - `Bool/SInt/UInt/Float/Double` are straightforward
  - `String` copies the string, not just the pointer
  - `Blob` copies a chunk of data
  - `Opaque` copies the address
  - `Null` writes a special marker for NULL
  - `Enum` and `Bitmask` writes the value as well as the signatures
    - there is a cache mechanism to make sure the signatures are written once
    - the signature is written so the the trace can be dumped by an old
      version of `tracedump`?
  - `beginArray/beginElement/write*/endElement/endArray` for array
  - struct is similar to array
- But Some functions are complex
  - see `EglTracer` and `GlTracer`
- For `eglGetProcAddress`, the returned function pointers are wrapped to point
  to the one defined by the tracer
- For `eglSwapBuffers`, `glFlush`, and `glFinish`, a snapshot may be optionally
  made
- For functions that may use VBO (e.g. `glVertexPointer`),
  - It is recorded normally when a VBO is bound
  - Otherwise, it sets `__user_arrays` to true, call the real function, and
    return.  They will be recorded just before the drawing functions.
  - But for `glInterleavedArrays`, it writes proper enables/disables before
    return
- For functions that may use PBO,
  - same as VBO, but easier as the size is known
- For all draw functions,
  - they trace the user-pointer-using functions now if needed
    - very expensive!
  - Then they are recorded normally.  If there are indices, the indices are
    also recorded if no VBO.
- For `glLinkProgram`, `glBindAttribLocation` are written before the call to
  make sure the attribs are allocated the same slots when retracing
  - so that we don't need to map the slots in `glUniform` and others
- For `glMapBuffer`, `glUnmapBuffer`, `glMapBufferRange`, and
  `glFlushMappedBufferRange`
  - the mapped pointer and the attributes of the mapping are rememebered when
    mapping
  - a `memcpy` is written so that we know what the app writes and can replay it
    when retracing
- For functions such as `glFogf(GL_FOG_MODE, 9729.0f)`, identify the float as an
  enum and record `glFogf(GL_FOG_MODE, GL_LINEAR)`

## Retracer

- The retracer is an executable and can retrace a trace file
- To achieve "trace anywhere, retrace anywhere", the retracer abstracts the
  winsys API (GLX/EGL/WGL/AGL) by `glws`.
- `glretrace.py` is used to generate the retrace functions for GL
  - for functions that have no side effect (e.g., `glGetError`), they are no-op
  - for each function argument, they are extracted and generate something like
    - `GLint x; x = (call.arg(0)).toSInt();`
  - the real function is called with the extracted arguments.  The returned
    value is always ignored if exists
  - ...
- Types
  - `Literal`: GLboolean, GLint
  - `Enum`: GLenum
  - `Bitmask`: GLbitfield
  - `String`: `GLchar*`
  - `Pointer`: pointer to a known type
  - `Array`: pointer to an array of known type and size
  - `Blob`: pointer to a hunk of known size
  - `Handle`: a texture object id, a buffer object id, and etc.
    - requires a map
  - `Opaque`, `OpaquePointer`, `OpaqueArray`, `OpaqueBlob`: an opaque type
    - e.g. vertex pointer of `glVertexPointer`
    - generally non-extractable
- e.g. `glGenTextures(n, textures)`
  - n is a literal: `n = call.arg(0).toSInt();`
  - textures is a `Out(Array(GLtexture))` of size n:
    - first, an array of size n for GLuint is allocated
    - every GLtexture is toUInt() and saved in the array
    - because GLtexture is a handle, every element in the array is mapped

## Replace GLX by EGL

- The tracer does not depend on EGL
  - but the tracer calls `__getPublicProcAddress` for public symbols and
    `__getPrivateProcAddress` for extensions
  - make the former to `dlopen` libEGL
  - make the latter to call `eglGetProcAddress`
  - see `glproc.py` and compare `egltrace.py` and `glxtrace.py`
- The retracer depends on EGL
  - because of the use of EGL-based `glws`
