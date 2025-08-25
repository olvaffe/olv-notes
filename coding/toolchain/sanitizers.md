Sanitizers
==========

## ASAN

- `-fsanitize=address`
- <https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html#index-fsanitize_003daddress>
  - `-g`, `-Og`, and `-fno-omit-frame-pointer` for better output
- compiling with `-fsanitize=address`
  - tells the compiler to instrument memory access instructions to detect
    out-of-bound and use-after-free bugs
  - the instrumentation references symbols from the sanitizer runtime,
    `libasan`, which must be linked to at link time or `LD_PRELOAD`ed at
    runtime
- linking with `-fsanitize=address`
  - tells the compiler to link to `libasan`
  - gcc links to `libasan` dynamically by default
    - executables link to `libasan` dynamically unless `-static-libasan`
    - shared libraries link to `libasan` dynamically unless `-static-libasan`
      - `-fPIC` is needed or compiler error
      - `-static-libasan` only statically link some symbols
        - other symbols such as `__asan_init` must be a singleton and cannot
          be linked statically into a shared library; they are expected to be
          provided by shared `libasan` or by the executable
        - `-Wl,--no-undefined` causes linking error
  - clang links to `libasan` statically by default
    - executables link to `libasan` statically unless `-shared-libasan`
    - shared libraries link to `libasan` statically unless `-shared-libasan`
      - `-fPIC` is optional
      - only some symbols are statically linked
        - similar to gcc, symbols such as `__asan_init` must be singleton and
          cannot be linked statically into a shared library
- executable and shared libararies
  - if an executable is built with `-fsanitize=address`,
    - with `-static-libasan`, the executable provides asan symbols to itself
      and to shared libraries
    - with `-shared-libasan`, shared `libasan` provides asan symbols to the
      executable and to shared libraries
  - if an executable is built without `-fsanitize=address`,
    - shared `libasan` must be `LD_PRELOAD`ed to provide asan symbols to
      shared libraries, or
    - `==594353==ASan runtime does not come first in initial library list; you should either link runtime to your application or manually preload it with LD_PRELOAD.`
  - if a shared library is built with `-fsanitize=address`,
    - with `-static-libasan`, `-Wl,--no-undefined` cannot not be specified
    - with `-shared-libasan`, shared `libasan` provides asan symbols to the
      shared library (but still must be `LD_PRELOAD`ed)
- Meson
  - `-Db_sanitize=address`
  - if shared library and clang, `-Db_lundef=false`
- CMake
  - add `-fsanitize=address` to `CMAKE_C_FLAGS` and `CMAKE_EXE_LINKER_FLAGS`
- Environment Variables
  - `ASAN_OPTIONS=help=1`
  - `ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer` to sepcify an external
    symbolier

## Rust

- requires nightly toolchains
- `RUSTFLAGS="-Z sanitizer=address" cargo build`
