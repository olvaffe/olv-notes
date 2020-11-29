Android RenderScript
====================

## Modules

* In `external/llvm`,
  * `libLLVM*.a` are built for both host and device
  * `llvm-as` and `llvm-link` are built for host
* In `external/clang`,
  * `libclang*.a` are built for host
  * `clang` is built for host
* In `frameworks/compile/linkloader`,
  * `librsloader.a` is built for both host and device
    * it is used by `libbcc.so`
* In `frameworks/compile/libbcc`,
  * `libbcc*.a` are built for both host and device
  * `libbcc.so` is built for both host and device
  * `libbccinfo.so` is built for both host and device
  * `libclcore.bc` is built for device
    * it is built from C sources using `clang` and `llvm-link`
  * `bcinfo` is built for host
* In `frameworks/compile/slang`,
  * `libslang.a` is built for host
    * it is used by `llvm-rs-link` and `llvm-rs-cc`
  * `librslib.a` is built for host
    * it is used by `llvm-rs-link`
  * `rs-spec-gen` is built for host
    * it is used to generate `.inc` files for `llvm-rs-cc`
  * `llvm-rs-link` is built for host
  * `llvm-rs-cc` is built for host
  * `slang-data` is built for host
    * not used?
* In `frameworks/base/lib/rs`,
  * `rsg-generator` is built for host
    * it is used to generate sources for `libRS.so`
  * `libRS.so` is built for device
  * `libRS.a` is built for host
    * not used?
* My guess
  * `slang` (`llvm-rs-cc` and `llvm-rs-link`) is the host-side tools to
    generate `.bc` and `.java` from `.rs`
  * `libbcc.so` is the JIT engine on device
  * `libRS.so` is RenderScript runtime

## API

* The API is described in `frameworks/base/libs/rs/scriptc`
