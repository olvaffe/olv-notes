CMake
=====

## cross-compile

- `32bit.cmake`
    set(CMAKE_SYSTEM_NAME Linux)
    set(CMAKE_C_COMPILER gcc)
    set(CMAKE_C_FLAGS -m32)
    set(CMAKE_CXX_COMPILER g++)
    set(CMAKE_CXX_FLAGS -m32)
- `cmake -DCMAKE_TOOLCHAIN_FILE=32bit.cmake`
- `arm64.cmake`
    set(toolchain_prefix /usr/bin/aarch64-linux-gnu-)
    set(sysroot ...)
    
    set(CMAKE_SYSTEM_NAME Linux)
    set(CMAKE_SYSTEM_PROCESSOR arm)
    
    set(CMAKE_SYSROOT ${sysroot})
    set(CMAKE_STAGING_PREFIX ...)
    
    set(CMAKE_C_COMPILER ${toolchain_prefix}gcc)
    set(CMAKE_CXX_COMPILER ${toolchain_prefix}g++)
    
    set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
    set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
