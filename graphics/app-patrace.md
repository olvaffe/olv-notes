PATrace
=======

## Use

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

## Android

- build
  - `apt install qtbase5-dev openjdk-8-jdk`
    - the build script uses `x11_x64` target to generate some files and
      requires qt5
    - it also requires java8
      - `update-alternatives` for `javac`
  - `./cmdline-tools/latest/bin/sdkmanager --install "platform-tools" "platforms;android-30" "build-tools;30.0.3" "ndk;21.1.6352462"`
    - to get android deps
  - `PATH=~/android/sdk/tools:$PATH ANDROID_SDK_ROOT=~/android/sdk NDK=~/android/sdk/ndk/25.1.8937393 ./scripts/build.py patrace android release`
- install
  - `adb install -r ./install/patrace/android/debug/eglretrace/eglretrace-release.apk`
  - install the gles layer
    - `adb shell mkdir /data/local/debug/gles`
    - `adb push ./install/patrace/android/release/gleslayer/libGLES_layer_arm64.so /data/local/debug/gles`
    - `adb push ./install/patrace/android/release/gleslayer/libGLES_layer_arm.so /data/local/debug/gles`
- use
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
