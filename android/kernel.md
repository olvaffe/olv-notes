Android Kernel
==============

## Build

- <https://source.android.com/docs/setup/build/building-kernels>
- `repo init -u https://android.googlesource.com/kernel/manifest -b BRANCH`
  - android15 ack uses `common-android15-6.6`
  - there is also `common-android15-6.6-desktop` for desktop
- `tools/bazel build //common:kernel_aarch64`
  - the binaries are under `bazel-bin/common/kernel_aarch64`
- manifest
  - `build/` has bazel rules to build the kernel
  - `common/` is the ACK kernel source code
  - `common-modules/` is the ACK kernel module source code
  - `external/` consists of external projects such as bazel, libcap, lz4,
    python, etc.
  - `kernel/` for configs and tests
  - `prebuilt/` consists of asuite, build-tools, clang, gcc, jdk, ndk, rust,
    tradefed, etc.
  - `tools/` has mkbootimg
