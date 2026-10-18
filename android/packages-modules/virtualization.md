# AVF

## Overview

- <https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/heads/android17-release/>
  - `android` is host-side (android) services, apps, and cli
    - `TerminalApp` is the terminal app for debian guest
    - `virtmgr` spawns `crosvm`
      - it talks to `virtualizationservice` for privileged ops
    - `virtualizationservice` is a privileged service
    - `vm` is a cli
  - `build` builds apex, debian img, and microdroid img
  - `guest` is guest-side pvm fw, services, and cli
    - `kernel` is prebuilt guest kernels
    - `linux_vm_manager` runs in debian guest
    - `microdroid_manager` runs in microdroid guest
  - `libs` is shared libraries and aidls (mostly for host and microdroid)

## CLI: `/apex/com.android.virt/bin/vm`

- `vm info`
  - it supports protected VMs and unprotected VMs
    - there is pkvm managing VMs
    - android itself runs inside a VM
    - android can poke unprotected VMs but not protected VMs
  - `microdroid` is a VM image based on android
- `vm run-microdroid` starts a VM using `microdroid` image
  - `--protected` starts a protected instead of an unprotected VM
  - `--mem 1024` allocs 1GB ram to the VM
  - `--debug none` uses production ramdisk and disables adbd inside VM
  - `--ephemeral` disables secretkeeper integration
  - the work dir is `/data/local/tmp/microdroid/<random>`
    - `storage.img` is a disk image for storage

## Terminal: `com.android.virtualization.terminal`

- pstree of `com.android.virtualization.terminal`
  - `virtmgr_zation.terminal`
    - `crosvm device fs` shares `/storage/emulated/10`
    - `crosvm device snd` emulates snd
    - `crosvm_debian` is the VM
- outside `crosvm_debian` VM
  - `/data/misc/virtualizationservice/<cid>` seems to be the temp dir
  - `/data/user/10/com.android.virtualization.terminal/files/linux` seems to
    be the vm data
    - `vm_config.json` defines the kernel, initrd, cmdline, rootfs, etc.
    - `root_part` is the sparse rootfs, `/dev/vda1`
    - `cidata.iso` is extra non-debian tools, `/dev/vdb`
      - it seems to be cloud-init NoCloud datasource
- inside `crosvm_debian` VM
  - standard debian
    - systemd{,-udevd,-networkd,-resolved,-logind,...}
    - dbus, pulseaudio
    - weston
  - non-standard debian
    - android gki
    - `ttyd`
    - `linux_vm_manager`
  - env
    - `DISPLAY=:0`
    - `LIBGL_ALWAYS_SOFTWARE=1`
    - `MESA_LOADER_DRIVER_OVERRIDE=zink`
    - `VK_DRIVER_FILES=/usr/share/vulkan/icd.d/lvp_icd.json`
- <https://android.googlesource.com/platform/packages/modules/Virtualization/+/refs/heads/android17-release/build/debian/>
  - `build.sh` builds a builder image and starts a container to build `images.tar.gz`
    - `build_internal.sh` runs inside container
      - `fetch_debian_image` downloads and unpacks `disk.raw` from official
        `debian-13-genericcloud-${DEBIAN_ARCH}.tar.xz`
        - <https://cloud.debian.org/images/cloud/>
      - `compile_ttyd` builds ttyd
        - <https://github.com/tsl0922/ttyd>
      - compile gui
        - <https://github.com/google/sommelier-rs>
        - <https://github.com/Supreeeme/xwayland-satellite>
        - <https://android.googlesource.com/platform/external/mesa3d>
      - `customize_rootfs`
        - it extracts `root_part` from `disk.raw`
        - `chroot_rootfs.sh`
          - `mount_environment` mounts `root_part` and others
          - it invokes `build_rootfs_in_chroot.sh`
        - `build_rootfs_in_chroot.sh` runs inside `root_part` chroot
          - `remove_default_kernels` purges unneeded packages
          - `install_packages` installs `GUEST_PACKAGES`
          - `install_ttyd` installs `/usr/local/bin/ttyd`
          - install gui
          - `apply_custom_configs` applies configs
      - `generate_final_package` packs `images.tar.gz`
        - `build_id`
        - `vm_config.json`
        - `vmlinuz` and `initrd.img`
          - <https://ci.android.com/builds/branches/aosp_kernel-common-android16-6.12/status.json>
          - `last_known_good_build` of `aosp_kernel-common-android16-6.12.kernel_$TARGET_ARCH`
        - `root_part`
  - `build_cidata.sh` builds `debian_cidata_$ARCH.iso`
    - `cidata-iso-package.go`'s `GenerateAndroidBuildActions`
      - `CopySpecsToDir` copies `linux_vm_manager`
      - `cp -R cloud-init_config ...`
      - generate iso
  - runtime
    - host: `TerminalApp` translates `vm_config.json` to crosvm args
      - `ds=nocloud` tells `cloud-init` to use `NoCloud` data source
      - `cidata.iso` is used as `/dev/vdb`
    - guest: `cloud-init` uses `cloud-init_config` to provision
      - `meta-data` defines `instance-id` that is generated at `cidata.iso`
        build time
      - `vendor-data` is empty
      - `user-data`
        - it creates user `droid`
        - it mounts `/dev/vdb` to `/mnt/cidata` to run `init.sh`
      - `network-config` defines netplan v2 config
