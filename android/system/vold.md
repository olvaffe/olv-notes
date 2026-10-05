# Android vold

## Mounts

- physical partitions
  - `misc` contains configs for a/b, normal/recovery, etc.
  - `boot_a` contains the gki kernel
  - `vbmeta_a` is for verified boot
  - `init_boot_a` contains the generic initramfs (`/init`, etc.)
  - `vendor_boot_a` contains the vendor initramfs (fstab, essential kernel
    modules, etc.)
  - `super` contains the ro android system
    - `system_a`
    - `system_ext_a`
    - `system_dlkm_a`
    - `vendor_a`
    - `vendor_dlkm_a`
    - `product_a`
  - `userdata` contains the rw user data
  - `metadata` is for verified and encrypted `userdata`, etc.
- first-stage `init` in initramfs mounts physical partitions
  - it uses `fs_mgr` to parse fstab
  - it sets up dm-linear devices over `super`
    - `system` to `/system` and then `switch_root`
    - `system_ext` to `/system_ext`
    - `system_dlkm` to `/system_dlkm`
    - `vendor` to `/vendor`
    - `vendor_dlkm` to `/vendor_dlkm`
    - `product` to `/product`
  - `metadata` to `/metadata`
- second-stage `init` mounts more partitions
  - `fs_mgr_mount_all` invokes `vdc cryptfs mountFstab`
    - it sets up a dm-default-key device over `userdata`
      - `userdata` to `/data`
  - `do_installkey` invokes `vdc cryptfs enablefilecrypto`
    - it enables file-based encryption for `/data`

## apk data

- `/data/app/~~<random>==/<package>-<random>==` is for installed apk
- `/data/data/<package>` is the internal private data dir of an app for user 0
- `/data/user/<user>/<package>` is the internal private data dir of an app
  - `/data/user/0` is a bind-mount of `/data/data`
  - no permission required to access
- `/data/media/<user>/Android/data/<package>` is the external private data dir of an app
  - `/mnt/user/<user>/emulated` is a fuse-mount of `/data/media`
    - it is like a bind-mount plus access control
    - modern app
      - with `MANAGE_EXTERNAL_STORAGE`: full access to `/mnt/user/<user>/emulated/<user>`
      - without: full access to `/mnt/user/<user>/emulated/<user>/Android/data/<package>`
    - legacy app
      - with `{READ,WRITE}_EXTERNAL_STORAGE`: read/write access to `/mnt/user/<user>/emulated/<user>`
      - without: no access
  - `/storage/emulated` is a bind-mount of `/mnt/user/<user>/emulated`
    - it relies on mount namespace to point to current user
  - `/sdcard -> /storage/self/primary -> /storage/emulated/<user>` are symlinks
    - it relies on mount namespace to point to current user
