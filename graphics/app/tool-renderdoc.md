RenderDoc
=========

## Build

- `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DENABLE_QRENDERDOC=off -DENABLE_PYRENDERDOC=off`
  - to enable `qrenderdoc`,
    - `apt install python3-dev qtbase5-dev libqt5svg5-dev libqt5x11extras5-dev`
  - to disable GL/GLES,
    - `-DENABLE_GL=OFF`, `-DENABLE_GLES=OFF`, and `-DENABLE_EGL=OFF`
- `ninja install`, or
  - edit `renderdoc_capture.json` to point to local `librenderdoc.so`
  - copy `renderdoc_capture.json` to `~/.local/share/vulkan/implicit_layer.d`

## Cross-Compile

- THIS DOES NOT WORK.  USE REMOTE SERVER INSTEAD.
- build qt
  - `wget https://download.qt.io/archive/qt/5.15/5.15.3/single/qt-everywhere-opensource-src-5.15.3.tar.xz`
  - `PKG_CONFIG_LIBDIR=... ../configure -extprefix `pwd`/aaa -opensource
       -confirm-license -release -static -xplatform linux-aarch64-gnu-g++
       -sysroot <sysroot> -opengl es2 -xcb-xlib -xcb -nomake tests
       -nomake examples -nomake tools -skip qtwebengine -skip qtwebglplugin`
    - look for `xcb_syslibs` in `qtbase/src/gui/configure.json`, which hints
      the dependencies for x11extras
    - I need to install dependencies in both host and sysroot.  Something is
      not right.
  - `make` and `make install`
- configure renderdoc with `-DSTATIC_QRENDERDOC=on` and
  `-DQT_QMAKE_EXECUTABLE=<path-to-qmake>`
  - edit `qrenderdoc/CMakeLists.txt` such that `SWIG_CONFIGURE_CC` and
    `SWIG_CONFIGURE_CXX` use the host compilers

## `renderdoccmd`

- capture
  - `renderdoccmd capture -c <output-name> -w <executable> ...`
  - might want `-d` to specify the working dir
- replay
  - `renderdoccmd replay <capture.rdc>` just works
  - `renderdoccmd remoteserver` just works for remote replay
