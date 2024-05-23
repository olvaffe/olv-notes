Android Chroot
==============

## Create Android Chroot

- unpack `system.img` to `chroot/`
- unpack `vendor.img` to `chroot/vendor`
- manually bind apex
  - `jar -xf com.android.runtime.apex apex_payload.img`
  - unpack `apex_payload.img` to `chroot/apex/com.android.runtime`
- bind mounts
  - `mount --rbind /sys chroot/sys`
  - `mount --rbind /dev chroot/dev`
  - `mount --rbind /proc chroot/proc`
- `sudo chroot chroot /bin/sh`
- debug
  - copy statically-linked busybox to chroot/ and
    `sudo chroot chroot /busybox sh`
  - build statically-linked strace for debugging
    - `LDFLAGS=-static ./configure --enable-mpers=no`

## Bionic Initialization

- the dynamic linker, `/system/bin/linker64`, is statically linked
  - after it is loaded by the kernel, the kernel jumps to `_start` which calls
    `__libc_init` in `libc_init_static.cpp`
  - `__libc_init_AT_SECURE` opens `/dev/null` and aborts on failure
  - `__libc_init_common`'s `__system_properties_init` attemps to open
    `/dev/__properties__` which is missing in chroot because we don't run
    `init`
- the linker calls `init_default_namespaces`
  - it attempts to open `/system/etc/ld.config.arm64.txt` and
    `/linkerconfig/ld.config.txt` which are both missing in chroot
- the linker logs with `async_safe_format_log_va_list`
  - it attempts connect to `/dev/socket/logdw` which is missing in chroot
- the linker starts loading each executable/library
  - it calls functions marked as `__attribute__((constructor))`
  - `__libc_preinit` is one of them
- after the linker loads the executable and all libraries, it jumps to
  `_start` of the executable.  `_start` calls `__libc_init` in bionic

## init

- this gives us properties, logcat, adb, etc.
- to make `systemd-nspawn` happy,
  - `touch <chroot>/system/etc/os-release`
  - `ln -sf /system <chroot>/usr`
  - `ln -sf /system/bin <chroot>/sbin`
  - `mkdir -p <chroot>/mkdir -p <chroot>/{tmp,run,var,var/log}`
  - `sudo cp /etc/localtime chroot/system/etc`
- `systemd-nspawn -bnUD <chroot>`
  - this does not work
  - `init` has first stage init and second stage init
  - first stage init is hardcoded and is hard to fool
  - maybe we can get lucky by skipping the first stage?
- maybe starting logd and adbd manually.  Not sure about properties.

## GSI image

- build image
  - <https://source.android.com/setup/build/gsi>
  - `repo init -u https://android.googlesource.com/platform/manifest -b android12-gsi`
  - `repo sync -c`
  - `. build/envsetup.sh`
  - `lunch gsi_x86_64-userdebug`
  - `make`
- create chroot
  - `mount -o ro system.img tmp`
  - `cp -a tmp chroot`
  - `umount tmp`
  - `mkdir chroot/usr`
    - this makes systemd-nspawn happy
- 
