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

## PATrace

- build
  - `git clone https://github.com/ARM-software/patrace.git`
  - `git submodule update --recursive --init`
  - `cmake -S patrace/project/cmake -B out -DCMAKE_TOOLCHAIN_FILE=toolchains/wayland_aarch64.cmake -DCMAKE_BUILD_TYPE=Debug`
    - the cmdline is derived from `./scripts/build.py patrace wayland_aarch64 debug`
    - fix the toolchain file
      - `SET(WINDOWSYSTEM x11)` for x11
    - remove `-Werror`
  - `make -C out`
- use
  - `LD_PRELOAD=/my/path/libegltrace.so app` to trace
  - `paretrace trace.1.pat` to replay
- tools
  - `-DENABLE_TOOLS=TRUE` to enable
  - `totxt` can convert trace to txt
- android build
  - `apt install qtbase5-dev openjdk-8-jdk`
    - the build script uses `x11_x64` target to generate some files and
      requires qt5
    - it also requires java8
      - `update-alternatives` for `javac`
  - `./cmdline-tools/latest/bin/sdkmanager --install "platform-tools" "platforms;android-30" "build-tools;30.0.3" "ndk;21.1.6352462"`
    - to get android deps
  - `PATH=~/android/sdk/tools:$PATH ANDROID_SDK_ROOT=~/android/sdk NDK=~/android/sdk/ndk/25.1.8937393 ./scripts/build.py patrace android release`
- android install
  - `adb install -r ./install/patrace/android/debug/eglretrace/eglretrace-release.apk`
  - install the gles layer
    - `adb shell mkdir /data/local/debug/gles`
    - `adb push ./install/patrace/android/release/gleslayer/libGLES_layer_arm64.so /data/local/debug/gles`
    - `adb push ./install/patrace/android/release/gleslayer/libGLES_layer_arm.so /data/local/debug/gles`
- android use
  - for tracing, see <https://developer.android.com/ndk/guides/rootless-debug-gles>
    - `adb shell settings put global enable_gpu_debug_layers 1`
    - `adb shell settings put global gpu_debug_layers_gles libGLES_layer_arm64.so`
    - `adb shell settings put global gpu_debug_app <package_name>`
    - the trace file will be written to `/data/apitrace/%s`
      - `%s` is `/proc/<pid>/cmdline` which is the package name
      - `adb shell mkdir -p /data/apitrace/%s`
      - `adb shell chmod 777 /data/apitrace/%s`
      - `chcon u:object_r:app_data_file:s0:c512,c768 /data/apitrace/%s`
  - for replay, `adb shell am start -n com.arm.pa.paretrace/.Activities.RetraceActivity --es fileName /absolute/path/to/tracefile.pat`
- fakedriver (without `GLESLAYER`)
  - it should be installed as the driver
  - when `glFoo` is called for the first time, it does
    `sp_glFoo = wrapper::CWrapper::GetProcAddress("glFoo")` and forward the
    call to `sp_gllFoo`
  - `LoadConfigFiles`
    - gets proc name by reading `/proc/<pid>/cmdline`
    - if the proc is listed in `/system/vendor/lib64/egl/fpsAppList.cfg` or
      `/system/lib64/egl/fpsAppList.cfg`, set `sShowFPS`
    - if the proc is listed in `appList.cfg`, set `sDoIntercept`
    - if `sDoIntercept`, read `interceptor.cfg` to `sInterceptorPath`
  - `GetProcAddress`
    - dlopens the interceptor or the real driver depending on whether the proc
      is traced
    - the interceptor is `sInterceptorPath`
    - the real driver is assumed to be
      - `libGLESv2_adreno.so`
      - `wrapped_libGLESv2.so`
      - `libGLESv2_mali.so`
    - uses `dlsym` and `eglGetProcAddress` to look up `glFoo`
- interceptor (without `GLESLAYER`)
  - the interceptor defines all egl/gl symbols
  - its `glFoo` calls its `patrace_glFoo`
  - `patrace_glFoo` does its things and looks up the real driver's `glFoo`
    from hardcoded paths such as
    - `/system/lib64/egl/libGLES_mali.so`
    - `/vendor/lib64/egl/libGLES_mali.so`
    - `/vendor/lib64/egl/lib_mali.so`
    - `/vendor/lib64/egl/libEGL_adreno.so`
- fakedriver (with `GLESLAYER`)
  - fakedriver can be built as gles layers
    - since Q, android supports gles layers
  - `AndroidGLESLayer_Initialize`
    - `layer_init_next_proc_address` initializes the dispatch table for the
      next layer
    - `layer_init_intercept_map` initializes the dispatch table for the
      interceptor
  - `AndroidGLESLayer_GetProcAddress`
    - when `glFoo` is queried, it returns `glesLayer_foo`
    - `glesLayer_foo` forwards the `glFoo` call to `patrace_Foo`
    - `patrace_Foo` has access to the dispatch table for the next layer
- skqp tracing
  - skqp uses only pbuffer and the trace is considered bad
    - mainly because `threads` in json header is empty
  - one fix is to modify `patrace/src/tracer/trace.py` to treat
    `eglCreatePbufferSurface` the same as `eglCreateWindowSurface`
