Android Kernel
==============

## Overview

- <https://source.android.com/docs/core/architecture/kernel>
  - historically, android combines LTS kernel with Android-specific patches to
    form Android Common Kernel (ACK)
  - Since 5.4, ACK is also known as Generic Kernel Image (GKI) and consists of
    - generic core kernel
    - GKI modules
      - they are built from the same source tree and have no abi issue
    - vendor modules
      - they may be built from a different source tree
      - GKI promises a stable abi, Kernel Module Interface (KMI), for vendor
        modules
- <https://source.android.com/docs/core/architecture/kernel/android-common>
  - `android-mainline` is the primary development branch
    - it merges in upstream mainline whenever a tag (rc or release) is added
    - it branches off whenever a release is declared an lts
  - each branch is named `android<ANDROID_RELEASE>-<KERNEL_VERSION>`
    - each branch promises a stable abi (KMI) for modules
  - Android 15 (2024, V) launches with 6.6 to 6.1
  - Android 14 (2023, U) launches with 6.1 to 5.10
  - Android 13 (2022, T) launches with 5.15 to 5.4
  - Android 12 (2021, S) launches with 5.10 to 4.19
  - Android 11 (2020, R) launches with 5.4 to 4.19
  - for upgrades, the kernel version might not need change
    - a device launched with android 11 and upgraded to android 15 can stay at
      4.19 kernel
- <https://source.android.com/docs/core/architecture/kernel/generic-kernel-image>
  - devices launching with android 11 / kernel 5.4, or later, must support GKI
  - devices launching with android 12 / kernel 5.10, or later, must use GKI
- <https://source.android.com/docs/core/architecture/kernel/modules>
  - GKI modules are built from the same source tree as the GKI kernel is
    - protected GKI modules
      - they are conceptually a part of core kernel and can use internal abi
      - they are built as modules only to save space
    - unprotected GKI modules
      - they are conceptually vendor modules and must use KMI abi
      - they can be overriden by real vendor modules
  - vendor modules may be built from any source tree
    - they must use KMI abi
- <https://source.android.com/docs/core/architecture/android-kernel-file-system-support>
  - supported fs: exfat, ext4, f2fs, fuse, incfs, vfat, erofs
  - virtual fs: debugfs, overlayfs, procfs, sysfs, tmpfs, tracefs

## Build

- <https://source.android.com/docs/setup/build/building-kernels>
  - <https://android.googlesource.com/kernel/common>
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
