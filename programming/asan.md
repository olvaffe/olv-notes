Address Sanitizer
=================

## C

- example
  - `CFLAGS=-fsanitize=address -g -O1 -fno-omit-frame-pointer`
  - `LDFLAGS=-fsanitize=address -g`
  - `cc -c $CFLAGS example.c`
  - `cc -o example $LDFLAGS example.o`
- compiling with `-fsanitize=address`
  - tells the compiler to instrument memory access instructions to detect
    out-of-bound and use-after-free bugs
  - the instrumentation references symbols from the sanitizer runtime,
    `libasan`, which must be linked to at link time or `LD_PRELOAD`ed at
    runtime
- linking with `-fsanitize=address`
  - tells the compiler to link to `libasan`
  - on gcc, this links dynamically unless `-static-libasan` is given
  - on clang, this links statically unless `-shared-libasan` is given
- shared libraries
  - when `-shared` is given, `-shared-libasan` works as usual but
    `-static-libasan` is ignored
  - for the shared libraries linked with `-static-libasan` to find the
    runtime,
    - the executable must be linked to `libasan`, dynamically or statically
    - or, `LD_PRELOAD` must be used

## Rust

- requires nightly toolchains
- `RUSTFLAGS="-Z sanitizer=address" cargo build`

## Environment Variables

- `ASAN_OPTIONS=help=1`
- `ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer` to sepcify an external
  symbolier
