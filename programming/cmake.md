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
