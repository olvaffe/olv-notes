Android AVB
===========

## Overview

- <https://chromium.googlesource.com/chromiumos/platform/vboot_reference/+/refs/heads/main/>
  - cros bootloader that supports avb
  - `vb2api_load_kernel` loads the kernel
    - `GptNextKernelEntry` iterates over `IsBootableEntry` partitions
      - that is, partition types that are cros kernel or avb vbmeta
    - `vb2_load_android` loads the android kernel
      - `avb_slot_verify` is called with `boot` and `vendor_boot` partitions
- `avb_slot_verify` loads and verifies requested partitions
  - in cros dev mode, `AVB_SLOT_VERIFY_FLAGS_ALLOW_VERIFICATION_ERROR` is set
    to allow verification errors
  - `load_and_verify_vbmeta` loads requested partitions and verifies against
    vbmeta
    - vbmeta is set up such that smaller partitions are verified directly and
      larger partitions rely on kernel dm-verity
    - if vbmeta has `AVB_VBMETA_IMAGE_FLAGS_VERIFICATION_DISABLED`,
      partition verification is skipped
  - if vbmeta has `AVB_VBMETA_IMAGE_FLAGS_VERIFICATION_DISABLED`,
    - cmdline has `root=PARTUUID=...` to point to the system partition directly
  - otherwise, `avb_append_options` builds the cmdline
    - `androidboot.vbmeta.foo=...`
    - depending on `AVB_VBMETA_IMAGE_FLAGS_HASHTREE_DISABLED`,
      `androidboot.veritymode` is `disabled` or `enforcing`
- fastboot
  - `--disable-verification` sets `AVB_VBMETA_IMAGE_FLAGS_VERIFICATION_DISABLED`
    - this disables direct verification for smaller partitions and kernel
      dm-verity for larger partitions
  - `--disable-verity` sets `AVB_VBMETA_IMAGE_FLAGS_HASHTREE_DISABLED`
    - this disables kernel dm-verity for larger partitions
