CMake
=====

## Use

- Basics
  - generate
    - `cmake -S. -Bout -GNinja`
  - variables
    - `cmake -Bout -LH` to list variables
    - `CMAKE_BUILD_TYPE`
  - build
    - `ninja -Cout -v`
    - `make -Cout VERBOSE=1`
  - debug
    - `cmake -Bout --trace`
- `cmake-commands`
  - these are more global and should be used with care
    - `add_compile_definitions`
    - `add_compile_options`
    - `add_link_options` 
    - these should not be used
      - `add_definitions`
      - `include_directories`
      - `link_directories`
      - `link_libraries`
  - use these instead
    - `target_compile_definitions`
    - `target_compile_features`
    - `target_compile_options`
    - `target_include_directories`
    - `target_link_libraries`
    - `target_link_options`
    - these should not be used
      - `target_link_directories`
  - find
    - `find_file`
    - `find_library`
    - `find_package`
    - `find_path`
    - `find_program`
- `cmake-variables` for user customizations
  - `CMAKE_BUILD_TYPE`
  - `CMAKE_<LANG>_COMPILER`
  - `CMAKE_<LANG>_FLAGS`
- `cmake-env-variables` for user customizations
  - `DESTDIR`
  - `LDFLAGS`
  - `VERBOSE`
  - `CC`
  - `CFLAGS`
  - `CXX`
  - `CXXFLAGS`
- `cmake-toolchains`
  - `--toolchain <path>` or `-DCMAKE_TOOLCHAIN_FILE=<path>`
  - `CMAKE_SYSTEM_NAME`
    - `Linux` or `Android`
    - `uname -s`
  - `CMAKE_SYSTEM_PROCESSOR`
    - `x86_64` or `i386` or `aarch64` or `arm`
    - `uname -m`
  - `CMAKE_SYSROOT`
  - `CMAKE_C_COMPILER`
  - `CMAKE_CXX_COMPILER`
  - `CMAKE_CXX_COMPILER`
  - `CMAKE_FIND_ROOT_PATH_MODE_*`
    - cmake finds under `CMAKE_SYSROOT`, `CMAKE_FIND_ROOT_PATH`, and host
      system root

## cross-compile

- `32bit.cmake`
    set(CMAKE_SYSTEM_NAME "Linux")
    set(CMAKE_C_COMPILER "gcc")
    set(CMAKE_C_FLAGS "-m32")
    set(CMAKE_CXX_COMPILER "g++")
    set(CMAKE_CXX_FLAGS "-m32")
- `cmake -DCMAKE_TOOLCHAIN_FILE=32bit.cmake`
- `arm64.cmake`
    set(toolchain_prefix "/usr/bin/aarch64-linux-gnu-")
    set(sysroot "...")
    
    set(ENV{PKG_CONFIG_SYSROOT_DIR} "${sysroot}")
    set(ENV{PKG_CONFIG_LIBDIR} "${sysroot}/usr/lib/pkgconfig:${sysroot}/usr/share/pkgconfig")
    
    set(CMAKE_SYSTEM_NAME "Linux")
    set(CMAKE_SYSTEM_PROCESSOR "aarch64")
    
    set(CMAKE_SYSROOT "${sysroot}")
    set(CMAKE_STAGING_PREFIX "...")

    set(CMAKE_C_COMPILER "${toolchain_prefix}gcc")
    set(CMAKE_CXX_COMPILER "${toolchain_prefix}g++")
    
    set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
    set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
