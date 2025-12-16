Misc Graphics Libraries
=======================

## Windowing

- <https://github.com/glfw/glfw>
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

## Scenes and Models

- <https://github.com/assimp/assimp>
- <https://github.com/syoyo/tinygltf>

## Textures

- <https://github.com/KhronosGroup/KTX-Software>
- <https://github.com/ARM-software/astc-encoder>
  - both encoding and decoding
- <https://github.com/nothings/stb>
  - image: jpg, png, gif, etc.
  - font: ttf
  - audio: ogg

## UI

- <https://github.com/ocornut/imgui>
  - build abstract gui and convert to 3D cmds (e.g., `VkCommandBuffer`)

## Math

- <https://github.com/g-truc/glm>
  - OpenGL Mathematics
  - header-only C++ math library
  - same naming convention as GLSL
  - superset of GLSL in functionality

## Memory

- <https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator>

## Rust

- <https://github.com/ash-rs/ash>
  - `generator`
    - `git submodule update --checkout` to check out `generator/Vulkan-Headers`
    - the executable parses `vk.xml` and generates these files under `ash/src`
      - `extensions_generated.rs` defines the dispatch tables for extensions
      - `tables.rs` defines the dispatch tables for core functions
      - `vk/aliases.rs` defines aliases (when an extension is promoted to core)
      - `vk/bitflags.rs` defines bitflags
      - `vk/const_debugs.rs` defines `fmt::Debug` for enums
      - `vk/constants.rs` defines consts such as `MAX_EXTENSION_NAME_SIZE`
      - `vk/definitions.rs` defines structs
      - `vk/enums.rs` defines enums
      - `vk/extensions.rs` defines extension names and PFN types
      - `vk/feature_extensions.rs` defines extension enums and flags
      - `vk/features.rs` defines core PFN types
      - `vk/native.rs` defines types generated from `.h` headers using bindgen
        - e.g., video headers
  - init
    - `Entry::load` loads `libvulkan.so`
      - it also inits `StaticFn`, `EntryFnV1_0`, and `EntryFnV1_1` dispatch
        tables
    - `Entry::try_enumerate_instance_version` enumerates instance version
    - `Entry::create_instance` creates an instance
      - it also inits `InstanceFnV1_0`, `InstanceFnV1_1`, and `InstanceFnV1_3`
        dispatch tables
    - `Instance::enumerate_physical_devices` enumerates physical devices
    - `Instance::create_device` creates a logical devices
      - it also inits `DeviceFnV1_0`, `DeviceFnV1_1`, `DeviceFnV1_2`, and
        `DeviceFnV1_3` dispatch tables
  - ext
    - `khr::swapchain::Device::new` creates a dispatch table for
      `VK_KHR_swapchain`
