# ChromeOS Portage

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
    - change KEYWORDS to `"*"`
    - `ebuild <package-name>.ebuild`
    - `egencache --update --repo <chromiumos|portage-stable> <package-name>`


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

## amd64-generic

- boot the image
  - note that test or dev images are required
  - `cros_vm --start --board amd64-generic`
  - or,

    ```bash
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
    ```

  - or create a qcow2 first
    - `qemu-img create -f qcow2 -b chromiumos_test_image.bin -F raw cros.qcow2`
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

## USE flags

- `USE=angle -egl` and `arc-mesa-virgl`
  - with `USE=-egl`, the package disables egl/virgl and becomes vk-only
  - with `USE=angle`, the package
    - ships `init.gpu.rc` with `setprop ro.hardware.egl angle`
    - creates `/vendor/lib64/libGLESv2_angle.so` to point to
      `egl/libGLESv2_angle.so` provided by the vendor image
- `USE=cross_domain_context`
  - `vm_concierge` will start arcvm with `--gpu vulkan=true,context-types=cross-domain:...`
