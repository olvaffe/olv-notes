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
  - Android 16 (2025, B) launches with 6.12 to 6.6
  - Android 15 (2024, V) launches with 6.6 to 6.1
  - Android 14 (2023, U) launches with 6.1 to 5.10
  - Android 13 (2022, T) launches with 5.15 to 5.4
  - Android 12 (2021, S) launches with 5.10 to 4.19
  - Android 11 (2020, R) launches with 5.4 to 4.19
  - for upgrades, the kernel version might not need change
    - a device launched with android 11 and upgraded to android 15 can stay at
      4.19 kernel
  - ACK KMI branch lifecycle
    - year N-1, Dec: upstream LTS
    - year N, Mar: feature freeze
    - year N, Jun: KMI freeze
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
- <https://android.googlesource.com/kernel/manifest/+/refs/heads/common-android16-6.12-desktop/default.xml>
  - `build` is the build rules
  - `common` is the aosp common kernel
  - `common-modules` is the aosp common modules
  - `external` is external dependencies
  - `kernel` is the aosp common kernel configs
  - `prebuilt` is the prebuilt toolchains and tools
  - `tools` is aosp build tools
  - `private` is vendor modules
    - <https://source.android.com/docs/setup/build/building-pixel-kernels>
    - <https://android.googlesource.com/kernel/manifest/+/refs/heads/android-gs-tegu-6.1-android15-d4/default.xml>
    - `devices/google/foo` is the build rules
    - `google-modules` is the vendor modules
- <https://source.android.com/docs/core/architecture/kernel/modules>
  - `tools/bazel run //common:kernel_aarch64_dist`
    - this builds GKI image and GKI modules that are common to all devices
    - GKI modules are either protected (mostly) or unprotected (rarely)
      - they are defined in `modules.bzl`
        - `_COMMON_UNPROTECTED_MODULES_LIST` lists unprotected modules
      - protected GKI modules are logically a part of GKI image
        - they are built as modules only because they are not critical to boot
        - they can use non-exported symbols from GKI image
        - they can export symbols
        - they cannot be overriden by vendor modules
      - unprotected GKI modules are logically vendor modules
        - they can only use exported symbols from GKI image and GKI protected
          modules
        - they cannot export symbols already exported by GKI image and GKI
          protected modules
        - they can be overriden by vendor modules
    - GKI image exports symbols defined in `gki/aarch64/abi.stg`
    - protected GKI modules exports symbols defined in
      `gki/aarch64/protected_exports`

## Kleaf

- <https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/README.md>
- `tools/bazel build //common:kernel_aarch64`
  - the bazel build file is `common/BUILD.bazel`
    - make goals is `_GKI_X86_64_MAKE_GOALS`, which is equivalent to
      `make bzImage modules`
  - the kernel config file is `common/arch/arm64/configs/gki_defconfig`
  - the output is `bazel-bin/common/kernel_aarch64/`
    - kernel image, modules, symbols, etc.
- `tools/bazel build //common:kernel_aarch64_dist`
  - various android images, such as `boot.img`, `system_dlkm.img`
  - packaged headers, such as `kernel-headers.tar.gz`
- `tools/bazel run //common:kernel_aarch64_dist`
  - this copies files to `out/kernel_aarch64/dist/`
- `tools/bazel build //private/devices/google/foo:foo`
  - the bazel build file is `private/devices/google/foo/BUILD.bazel`
    - make goals is `dtbs modules`, which is equivalent to `make dtbs modules`
  - the output is `bazel-bin/private/devices/google/foo/foo/`
    - dtbs, vendor modules
- `tools/bazel build //private/devices/google/foo:foo_dist`
  - various android images, such as `vendor_dlkm.img`
  - this also builds external modules
    - e.g., `private/google-modules/bar/BUILD.bazel`
- `tools/bazel run //private/devices/google/foo:foo_dist`
  - this copies files to `out/foo/dist/`
- for cros,
  - build kernel
    - `tools/bazel run //common:kernel_aarch64_dist --config=stamp -- --wipe_destdir`
    - `tools/bazel run //private/devices/google/foo:foo_dist -- --wipe_destdir`
  - if arm64, the kernel image and the dtbs are packaged to fit image
    - the fit image packs `out/kernel_aarch64/dist/Image` and
      `out/foo/dist/*.dtb`
  - update prebuilt kernel
  - build android
    - `m installclean`
    - `m dist`
- `BUILD.bazel`
  - `load("@bazel_skylib//rules:common_settings.bzl", "bool_flag")`
    - <https://github.com/bazelbuild/bazel-skylib/blob/main/rules/common_settings.bzl>
  - `load("@rules_pkg//pkg:install.bzl", "pkg_install")`
    - <https://github.com/bazelbuild/rules_pkg/blob/main/pkg/install.bzl>
  - `load("//build/kernel/kleaf:kernel.bzl", "kernel_build")`
    - <https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/kernel.bzl>
  - `package(...)`
    - <https://bazel.build/reference/be/overview>
  - `filegroup(name = "foo", srcs = ...)`
  - `kernel_build(...)`
    - <https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/docs/api_reference/kernel.md#kernel_build>
    - `outs` is a list of non-module outputs, such as dtbs
    - `build_config` is the kleaf build config file
    - `module_outs`  is a list of in-tree modules
    - `arch` is `arm64` or `x86_64`
    - `base_kernel` is empty, or, e.g., `//common:kernel_aarch64`
    - `make_goals` overrides build config `MAKE_GOALS`
      - base kernel should have `MAKE_GOALS="Image modules"`
      - vendor modules should override to `["modules", "dtbs"]`
    - `defconfig`, `pre_defconfig_fragments`, and `post_defconfig_fragments`
      - base kernel should have `defconfig = "arch/arm64/configs/gki_defconfig"`
    - `strip_modules` strips debug symbols from modules
  - `kernel_module_group(...)`
    - <https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/docs/api_reference/kernel.md#kernel_module_group>
    - `src` is a list of of `kernel_module` (external modules) or `ddk_module`
  - `kernel_modules_install(...)`
    - <https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/docs/api_reference/kernel.md#kernel_modules_install>
    - `kernel_build` is a `kernel_build`
    - `kernel_modules` is a list of `kernel_module`s
  - `system_dlkm_image(...)`
  - `vendor_dlkm_image(...)`

## Iterative Development

- `bazel run --config=fast ...`
  - specifically, avoid `--config=stamp`!
- copy kernel and modules to prebuilt
- `m bootimage system_dlkmimage vendorbootimage vendor_dlkmimage`
  - `bootimage` packs gki kernel to `boot.img`
  - `system_dlkmimage` packs gki modules to `system_dlkm.img`
  - `vendorbootimage` packs cmdline (`BOARD_KERNEL_CMDLINE`) and vendor
    ramdisk (including `BOARD_VENDOR_RAMDISK_KERNEL_MODULES`) to
    `vendor_boot.img`
  - `vendor_dlkmimage` packs vendor modules to `vendor_dlkm.img`
- `fastboot flash <part> <image>`
  - `part` is `boot`, `system_dlkm`, `vendor_boot`, and `vendor_dlkm`
    respectively
  - note that with dynamic partitions (`super.img`), dlkm can also be flashed
    through fastbootd
