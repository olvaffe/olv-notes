GLFW
====

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

## GLM

- OpenGL Mathematics
- header-only C++ math library
- same naming convention as GLSL
- superset of GLSL in functionality
