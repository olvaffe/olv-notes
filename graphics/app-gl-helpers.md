Open GL (ES) Helpers
====================

## GLEW

- handling GL function pointers
  - old
- usage
  - `glewInit()`
  - `if (GLEW_ARB_vertex_program) ...`
- build options
  - `GLEW_NO_GLU`, do not include glu.h
  - `GLEW_STATIC`, static library
  - `GLEW_EGL`, use EGL (`eglGetProcAddress`) instead of GLX
    (`glXGetProcAddress`)
- link against libEGL, libGL/libX11, or libOpenGL directly

## epoxy

- handling GL function pointers
  - newer
- usage
  - `#include <epoxy/gl.h>`
  - `#include <epoxy/glx.h>`
  - `#include <epoxy/egl.h>`
- build options
  - `glx`, enable GLX support
  - `egl`, enable EGL support
  - `x11`, control GLX or EGL-X11 support
- no direct linking

## GLFW

- simple windows, contexts, surfaces, inputs support
- usage
  - `glfwInit`
  - `glfwCreateWindow`
  - `glfwMakeContextCurrent(win)` to make the internal gl context current
  - `glfwCreateWindowSurface` to create a `VkSurfaceKHR` from the window
- build options
  - `BUILD_SHARED_LIBS`, build shared library
  - `GLFW_BUILD_X11`, support X11 with libX11 internally
  - `GLFW_BUILD_WAYLAND`, support wayland
- no direct linking
