Chrome OS Build
===============

## Build image

- Relax sudo
  - /etc/sudoers.d/relax_requirements
      Defaults !tty_tickets
      Defaults timestamp_timeout=180
- `cros_sdk` to enter chroot
- "export BOARD=<BOARD-NAME>"
- switch kernel branch, if needed
- `USE="kgdb vtconsole" ./build_packages --board=${BOARD}`
- `./build_image \
    --boot_args="noinitrd slub_debug=FZPUA panic=0" \
    --board=${BOARD} \
    --noenable_rootfs_verification \
    test`
- internals
  - `build_packages`
    - it builds and emerges these packages by default to /build/$BOARD
      - virtual/target-os, base OS image
      - virtual/target-os-dev, developer tools (shell, ssh, vim, tcpdump, etc)
      - virtual/target-os-factory, factory tools
      - virtual/target-os-factory-shim, factory installer (flashrom, etc)
      - virtual/target-os-test, deqp, etc.
      - chromeos-base/autotest-all
    - binary packages can be found under /build/$BOARD/packages
    - before it builds, it invokes `update_chroot --toolchain_boards $BOARD`
      - `Updating cross-compilers` invokes `cros_setup_toolchains` to update
      	chroot toolchains
      - `Bootstrapping depot_tools`
      - `Rebuilding Portage cache`
      - `Updating the SDK` builds chroot `virtual/target-sdk` and `world`
        - this steps easily takes >10m
  - `build_image`
    - it builds images under ~/trunc/src/build/$BOARD
    - it always wipes the build directory clean, install packages, and done

## Manual Building

- `FEATURES="noclean" emerge-$BOARD $PACKAGE`
  - this leaves `/build/$BOARD/tmp/portage/*/$PACKAGE-*` on disk
  - `temp/environment` gives how the package is built
- Meson-based packages
  - get cross file from `temp/meson.x86_64-cros-linux-gnu.amd64.ini`
- Rust-based packages
  - get cargo config file from `work/cargo_home/config` and save it to
    `.cargo/config.toml`
  - might additionally need `RUSTFLAGS` and `PKG_CONFIG`
- autotools
  - SYSROOT="/build/$BOARD"
  - "autoreconf -f -v -i -I $SYSROOT/usr/share/aclocal"
  - TARGET="arch-vendor-sys-abi"
  - PATH adds "$SYSROOT/build/bin"
  - PKG_CONFIG points to "$SYSROOT/build/bin/pkg-config"
  - CC="$TARGET-clang"
  - CXX="$TARGET-clang"
  - CPPFLAGS="--sysroot=$SYSROOT"
  - LDFLAGS="--sysroot=$SYSROOT"
  - CFLAGS and CXXFLAGS add "-march=... -mtune=... ..."

## Portage

- check out board-specific /build/$BOARD/etc/make.conf
- src/overlays/chipset-qc845/profiles/base/make.defaults
  - `toolchain.conf` for toolchain
  - `CHROMEOS_KERNEL_SPLITCONFIG="chromiumos-qualcomm"`
  - `CHROMEOS_KERNEL_ARCH="arm64"`
  - `BOARD_COMPILER_FLAGS="-march=armv8-a+crc -mtune=cortex-a57.cortex-a53 -mfpu=crypto-neon-fp-armv8 -mfloat-abi=hard"`
- update package
  - `cros_portage_upgrade --upgrade --board=amd64-generic:arm-generic <package-name>`
  - manual
    - add new ebuild
    - change KEYWORDS to "*"
    - ebuild <package-name>.ebuild
    - egencache --update --repo <chromiumos|portage-stable> <package-name>

## Debug

- On DUT
  - PORT=1234
  - use ssh forwarding
    - ssh -L $PORT:localhost:$PORT
    - or allow debug port if blocked
      - iptables -A INPUT -p tcp --dport $PORT -j ACCEPT
      - iptables -L INPUT
  - gdbserver :$PORT <program>
- On host
  - $TARGET-gdb \
       -ex "set sysroot /build/$BOARD" \
       -ex "set solib-absolute-prefix /build/$BOARD" \
       -ex "set debug-file-directory /build/$BOARD/usr/lib/debug" \
       -ex "set directories $SRC_DIRS" \
       -ex "target remote :$PORT"
  - libs being developed needs to be copied to /build/$BOARD for $TARGET-gdb to pick them up?
    - $TARGET-objcopy --only-keep-debug $lib /build/$BOARD/usr/lib/debug/usr/lib/$lib.debug
    - $TARGET-objcopy --strip-debug \
        --add-gnu-debuglink=/build/$BOARD/usr/lib/debug/usr/lib/$lib.debug \
        $lib /build/$BOARD/usr/lib/$lib
    - scp /build/$BOARD/usr/lib/$lib $DUT:/usr/lib/$lib

## amd64-generic

- boot the image
  - note that test or dev images are required
  - `cros_vm --start --board amd64-generic`
  - or,
    $ qemu-system-x86_64 \
        -cpu SandyBridge,-invpcid,-tsc-deadline,check,vmx=on \
        -accel kvm -smp 4 -m 8G \
        -device virtio-vga \
        -device virtio-scsi-pci -device scsi-hd,drive=my-disk \
        -drive if=none,id=my-disk,file=chromiumos_test_image.bin,cache=unsafe,format=raw \
        -device virtio-net,netdev=my-net \
        -netdev user,id=my-net,hostfwd=tcp::2222-:22 \
        -device virtio-rng \
        -usb -device usb-tablet
   - or
     - create a qcow2 first
       `qemu-img create -f qcow2 -b chromiumos_test_image.bin -F raw cros.qcow2`
- profile inheritance tree
  - ordering is depth-first, left-to-right
  - `overlays/overlay-amd64-generic/profiles/base/parent`
    - `third_party/chromiumos-overlay/profiles/default/linux/amd64/10.0/chromeos/parent`
      - `third_party/chromiumos-overlay/profiles/default/linux/amd64/10.0/parent`
        - `third_party/chromiumos-overlay/profiles/default/linux/amd64/parent`
          - `third_party/chromiumos-overlay/profiles/base`
          - `third_party/chromiumos-overlay/profiles/default/linux`
          - `third_party/chromiumos-overlay/profiles/arch/amd64/parent`
            - `third_party/chromiumos-overlay/profiles/arch/base`
            - `third_party/chromiumos-overlay/profiles/features/multilib/lib32/parent`
              - `third_party/chromiumos-overlay/profiles/features/multilib`
        - `third_party/chromiumos-overlay/profiles/releases/10.0/parent`
          - `third_party/chromiumos-overlay/profiles/releases`
      - `third_party/chromiumos-overlay/profiles/targets/chromeos`
      - `third_party/chromiumos-overlay/profiles/features/llvm/amd64/parent`
        - `third_party/chromiumos-overlay/profiles/features/llvm`
    - `third_party/chromiumos-overlay/profiles/features/selinux`
- `setup_board -b amd64-generic --profile fuzzer`
  - `overlays/overlay-amd64-generic/profiles/fuzzer/parent`
    - `overlays/overlay-amd64-generic/profiles/base/parent`
      - see above
    - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer/asan/amd64/parent`
      - `third_party/chromiumos-overlay/profiles/features/sanitizers/asan/amd64/parent`
        - `third_party/chromiumos-overlay/profiles/features/sanitizers/asan/parent`
          - `third_party/chromiumos-overlay/profiles/features/sanitizers`
      - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer/asan/parent`
        - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer`
- `./build_packages` builds these packages by default
  - `virtual/target-os`
    - this depends on `virtual/target-chromium-os` which includes a long list
      of packages
  - `virtual/target-os-dev`
  - `virtual/target-os-factory`
  - `virtual/target-os-factory-shim`
  - `virtual/target-os-test`
  - `chromeos-base/autotest-all`
- these packages depend on `virtual/opengles` when `USE=opengles`
  - `chromeos-base/chromeos-chrome`
  - `chromeos-base/drm-tests`
  - `chromeos-base/glbench`
  - `dev-util/apitrace`
  - `media-gfx/deqp`
  - `media-libs/libepoxy`
  - `media-libs/waffle`
  - maybe more
- these packages depend on `media-libs/vulkan-loader` when `USE=vulkan`
  - `chromeos-base/drm-tests`
  - `chromeos-base/vkbench`
  - `dev-util/vulkan-tools`
  - `media-libs/virglrenderer`
  - `virtual/vulkan-icd`
  - maybe more
- these packages depend on `virtual/vulkan-icd` when `USE=vulkan`
  - `chromeos-base/drm-tests`
  - `chromeos-base/vkbench`
  - `media-gfx/deqp`
  - maybe more
- virtualization
  - `USE=kvm_host` means enable KVM support in the host kernel
    - `target-chromium-os` uses the flag to add `crostini_client`,
      `vm_host_tools`, and `termina-dlc`
    - `vm_host_tools` depends on `crosvm`
  - when `USE=crosvm-gpu`, `crosvm` depends on `virglrenderer`
  - `USE=kvm_guest` means building for the (crostini) container image

## gdb

- must use gdbserver
  - gdbserver uses hw breakpoints
  - gdb uses sw breakpoints
  - because cros kernel is hardened, sw breakpoints do not work
- debug locally-built image
  - `/build/$BOARD` is the sysroot
  - `/build/$BOARD/usr/lib/debug` has the debug symbols
  - `aarch64-cros-linux-gnu-gdb -ex 'target remote :1234' \
       -ex 'set sysroot /build/$BOARD' \
       -ex 'set debug-file-directory /build/$BOARD/usr/lib/debug'`
- debug released image
  - download `chromiumos_test_image.tar.xz` and `debug.tgz`
  - unpack `debug.tgz` to `debug`
  - unpack `chromiumos_test_image.tar.xz`
  - `./mount_gpt_image.sh -f <dir> -i chromiumos_test_image.bin`
    - this mounts the various paritions in the disk image to `/tmp/m`
    - rootfs is rw
      - cros sets undefined features flags of ext4 to force read-only rootfs
      - `enable_rw_mount` clears those flags and allows the rootfs to be
        mounted rw
      - see `build_library/ext2_sb_util.sh`
  - bind mount debug symbols (this is unnecessary...)
    - `rm /tmp/m/usr/lib/debug`
    - `mkdir /tmp/m/usr/lib/debug`
    - `mount --bind debug /tmp/m/usr/lib/debug`
  - `aarch64-cros-linux-gnu-gdb -ex 'target remote :1234' \
       -ex 'set sysroot /tmp/m' \
       -ex 'set debug-file-directory /tmp/m/usr/lib/debug'`
  - to umount,
    - `./mount_gpt_image.sh -f <dir> -i chromiumos_test_image.bin -u`
- mount `chromiumos_test_image.bin` manually
  - `losetup -P -f chromiumos_test_image.bin`
  - `mount -o ro /dev/loop0p3 sysroot`
  - `mount /dev/loop0p1 sysroot/mnt/stateful_partition`
  - `mount --bind sysroot/mnt/stateful_partition/dev_image sysroot/usr/local`
