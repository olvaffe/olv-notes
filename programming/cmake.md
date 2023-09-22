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
  - `CMAKE_<LANG>_COMPILER_LAUNCHER`
    - e.g., ccache
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

## Dependencies

- <https://cmake.org/cmake/help/latest/guide/using-dependencies/index.html>
- `find_package` finds prebuilt packages
  - it searches well-known places, using hints provided by the project or
    users
  - it supports package components or optional packages
  - `find_package(Foo 3.4 REQUIRED COMPONENTS bar)`
- config mode
  - prebuilt packages that use cmake or are cmake-aware can provide necessary
    files for `find_package`
  - `find_package(Foo)` searches for `FooConfig.cmake` or `foo-config.cmake`
    in common paths such as `lib/cmake/Foo`
  - `CMAKE_PREFIX_PATH` var provides a semicolon-separated list of search
    paths
  - `CMAKE_PREFIX_PATH` envvar provides a colon-separated list of search paths
- module mode
  - `Find` modules must be provided separately for the prebuilt packages
  - `find_package(Foo)` searches for `FindFoo.cmake` in common paths
  - `CMAKE_MODULE_PATH` var provides a semicolon-separated list of search
    paths
- imported targets
  - both config and module modes may define imported targets
  - an imported target looks like `Foo::Bar`, and is preferred over other
    cmake variables provided
