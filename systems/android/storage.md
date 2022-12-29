Android Storage
===============

## Overview

- <https://android.googlesource.com/platform/system/core/+/refs/heads/master/rootdir/init.rc>
  - `on init`
    - create various dirs under `/mnt`
    - `symlink /storage/self/primary /mnt/sdcard`, for legacy apps
  - `on early-fs`
    - `start vold`
  - `on fs`: device-defined
  - `on post-fs`
    - `mount none /mnt/user/0 /storage bind rec`
  - `on late-fs`
  - `on late-fs`
  - `on post-fs-data`
    - create various dirs under `/data`
    - `mount none /data/data /data/user/0 bind rec`
- APIs
  - `getFilesDir` commonly returns `/data/user/0/<package_name>/files`
  - `getExternalFilesDirs` commonly returns
    `/storage/emulated/0/Android/data/<package_name>/files`
  - `getExternalMediaDirs` commonly returns
    `/storage/emulated/0/Android/media/<package_name>`
  - `getExternalStoragePublicDirectory` commonly returns a subdirectory under
    `/storage/emulated/0`
- my guess is
  - `/data/user/0` is a bind mount of `/data/data`
    - this is for app data of the default user
    - android can be multi-user
  - `/storage` is a bind mount of `/mnt/user/0`
    - this is for external storage of the default user
  - `/mnt/user/0` has subdirs used as mountpoints
    - `emulated/0` is an emulated external storage of the default user, using
      the internal storage
    - `self/primary` is a symlink to `emulated/0`
  - `/mnt/sdcard` is a symlink to `/storage/self/primary`

## Flash Partitioning

- the flash can be partitioned to
  - `boot_a` and `boot_b`
  - `dtbo_a` and `dtbo_b`
  - `recovery_a` and `recovery_b`
  - `vbmeta_a` and `vbmeta_b`
  - `metadata`
  - `userdata`
  - `super`
    - <https://source.android.com/devices/tech/ota/dynamic_partitions>
    - this replaces `system_*`, `vendor_*`, and `product_*` using `dm-linear`
- my guess
  - `boot_a` has the kernel and initramfs
    - `/init` does the boot setup and remount `/` as ro
  - `system_a` is ro mounted to `/system`
  - `vendor_a` is ro mounted to `/vendor`
  - `product_a` is ro mounted to `/product`
  - `metadata` is rw mounted to `/metadata`
  - `userdata` is rw mounted to `/data`
    - it is encrypted
    - `metadata` has credenticals to decrypt `userdata`
  - <https://source.android.com/devices/bootloader/partitions>
