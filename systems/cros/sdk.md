Chrome OS SDK
=============

## New Device

- enable developer mode
  - <https://chromium.googlesource.com/chromiumos/docs/+/HEAD/developer_mode.md#dev-mode>
  - if the device is corp-managed
    - log in using corp account first
    - `chrome://policy`, reload policies, and confirm `DeviceBlockDevMode` is
      false
  - hold `ESC` and `F3/Refresh`, then press power button to boot into recovery mode
    - `Power + Volume-Up + Volume-Down` for 10s if tablet
  - while in recovery mode, press Ctrl-D to enter developer mode
    - `Volume-Up + Volume-Down` if tablet
- select boot device
  - `Ctrl-D` to boot from disk
    - first boot will take 5-10 minutes
  - `Ctrl-U` to boot from USB
    - require `crossystem dev_boot_usb=1` first
  - `Ctrl-L` to chainload an alternative bootloader
    - require `crossystem dev_boot_altfw=1` first
- console
  - `Ctrl-Alt-F2` to enter console
- remove rootfs verification and enable SSH
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/chromeos-base/chromeos-sshd-init/files/openssh-server.conf.README>
  - use `Enable debugging features` on welcome screen
    - this removes rootfs verification and enables SSH
  - or manually
    - `/usr/share/vboot/bin/make_dev_ssd.sh -r`
    - `reboot`
    - `/usr/libexec/debugd/helpers/dev_features_ssh`
    - optionally `passwd`
      - or use <https://chromium.googlesource.com/chromiumos/chromite/+/refs/heads/main/ssh_keys/testing_rsa>
- flash latest test image
  - `cros flash -v --no-ping --disable-rootfs-verification ${DUT_IP} xbuddy://remote/${BOARD}/latest-canary/test`
    - sometimes `--clobber-stateful --clear-tpm-owner` is required to reformat
      the stateful partition
    - if forgot, `echo "fast safe" > /mnt/stateful_partition/factory_install_reset` to powerwash
  - or, create and flash from a bootable USB
    - `cros flash usb:// xbuddy://remote/${BOARD}/latest-canary/test`
    - Ctrl-U to boot from USB
    - `chromeos-install`
- write down firmware versions
  - H1 firmware: `gsctool -a -f`
  - EC firmware: `ectool version`
  - AP firmware: `crossystem fwid`
  - OS image: `grep CHROMEOS_RELEASE_DESCRIPTION /etc/lsb-release`
- follow <../../firmwares/cros.md> to update firmwares
- after updating firmwares, we might lose developer mode
  - we should see boot screen on boot when in dev mode
  - `crossystem devsw_boot` should return 1
  - if not, re-enter the developer mode

## Exit Developer Mode

- enable usb boot
  - if a dev-signed firmware is flashed, recovery mode does not work
  - it is possible to copy `chromeos-firmwareupdate` from a signed image and
    `chromeos-firmwareupdate -m factory --force` to flash a MP-signed firmware
    manually, but usb boot is simpler
- flash a signed image to usb
- boot from usb
  - this will flash both the mp-signed firmware and image
- return to secure mode
  - or not

## Get Source

- <https://chromium.googlesource.com/chromiumos/docs/+/main/developer_guide.md>
- `depot_tools`
  - `git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git`
  - `export PATH=/path/to/depot_tools:$PATH`
- get source code
  - `mkdir -p ~/chromiumos`
  - `cd ~/chromiumos`
  - `repo init -u https://chromium.googlesource.com/chromiumos/manifest -b main`
    - if googler, use <https://chrome-internal.googlesource.com/chromeos/manifest-internal>
    - also install `google-cloud-sdk`, `gcloud auth login`, and leave the
      project id at 0
    - because cros uses bundled `gsutil`, copy the credential 
      `cp ~/.config/gcloud/legacy_credentials/*/.boto ~/.boto`
  - `repo sync -j4`
- Source Tree Layout
  - `chromite`, build tools and scripts
  - `crostools`, image signing (internal)
  - `docs`, documentations
  - `infra`, CI, test infrastructure
  - `infra_virtualenv`, python virtualenv that infra depends on
  - `manifest`, repo manifest
  - `manifest-internal`, repo manifest (internal)
  - `website`, for website
  - `src`
    - `aosp`, random projects from AOSP
    - `chromium`, random projects from Chromium
    - `config`, config schemas that describe board hw and sw
    - `config-internal`, infra-related board configs (internal)
    - `overlays`, portage board overlays
    - `platform`, cros-specific platform projects
    - `platform2`, cros-specific platform projects, for new projects
    - `private-overlays`, portage board overlays (internal)
    - `program`, board sw/hw configs (internal)
    - `project`, board sw/hw configs (internal)
    - `repohooks`, git hooks
    - `scripts`, to build and install packages in chroot
    - `third_party`, tons of third-party projects
- Build Artifacts
  - `src/build`, `build_image` outputs
  - `devserver`, images served by dev server
- Portage
  - `src/third_party/portage-stable`, ebuilds copied from Gentoo portage-stable
  - `src/third_party/chromiumos-overlay`, ebuilds for Chromium OS-specific apps

## SDK chroot

- `cros_sdk` will download and enter SDK chroot
- to build an image,
  - `build_packages --board=BOARD --no-withdebug --internal`
  - `build_image --board=BOARD --no-enable-rootfs-verification test`
- SDK is a cros (gentoo) chroot tarball
  - <https://chromium.googlesource.com/website/+/HEAD/site/chromium-os/build/sdk-creation/index.md>
  - `cros_sdk` downloads the tarball from, for example,
    <https://storage.googleapis.com/chromiumos-sdk/cros-sdk-2022.12.03.124835.tar.xz>
  - it untars the tarball into `~/chromiumos/chroot`
  - conceptually, to build a SDK tarball,
    - `build_packages --board=amd64-host target-sdk`
- because SDK is a gentoo chroot, it can be updated without re-downloading the
  tarball
  - packages can be updated or installed by `emerge`
  - `./update_chroot` updates `target-sdk` package
    - this is invoked by `build_packages` automatically
- chroot directory layout
  - `$HOME/chromiumos`, symlink to `/mnt/host/source`
  - `/mnt/host/source`, bind-mount of the source tree
  - `/usr/local/bin`
  - `/build`, `build_packages` outputs
  - the rest are managed by `emerge`

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

## Flash Images

- <https://www.chromium.org/chromium-os/build/cros-flash>
- `cros flash <device> <image>`
  - `<device>` can be `usb://` or `ssh://<DUT-IP>` or `file://<local-dir>`
  - `<image>` is `latest` by default
  - for prod images, specify `--disable-rootfs-verification` to remove rootfs verification
- flash a locally built image
  - `cros flash ssh://<DUT-IP>`
    - or, `cros flash ssh://<DUT-IP> latest`
    - or, `cros flash ssh://<DUT-IP> ${BOARD}/latest`
  - these flash `~/trunk/src/build/images/$BOARD/latest`
- flash using a devserver
  - `cros flash ssh://<DUT-IP> xbuddy://remote/<<BOARD>/<VERSION>/<TYPE>`
  - `TYPE` can be
    - `test` (default) is `base` plus dev and test packages
      - `chromiumos_test_image.tar.xz`
    - `base` is the base image signed with dev key
      - `chromiumos_base_image.tar.xz`
    - `recovery` is the recovery image signed with dev key
      - `recovery_image.tar.xz`
      - this requires `dev_boot_usb=1`
    - `signed` is `recovery` signed with mp key
      - this is for use with the recovery mode
- flash using a usb disk
  - after flashing the image to a usb disk, boot from usb and run
  - `chromeos-install`
- download image and flash
  - `cros flash file://<path/to/file> xbuddy://remote/...`
  - `cros flash ssh://<DUT-IP> path/to/file`
- `cros flash -v ssh://<DUT-IP> xbuddy://remote/...`
  - gsutil cp `chromeos_*_full_dev.bin`, which is rootfs
  - scp to dut and flash
  - gsutil cp `stateful.tgz`, which is stateful partition
  - scp to dut, touch `/mnt/stateful_partition/.update_available`, and reboot
- disable rootfs verification afterwards
  - untested
  - `crossystem dev_boot_signed_only=0`
  - `/usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification`

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
       -ex 'set debug-file-directory /build/$BOARD/usr/lib/debug' \
       -ex 'set sysroot /build/$BOARD'`
    - remember to set `debug-file-directory` before `sysroot`
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
       -ex 'set debug-file-directory /tmp/m/usr/lib/debug' \
       -ex 'set sysroot /tmp/m'`
  - to umount,
    - `./mount_gpt_image.sh -f <dir> -i chromiumos_test_image.bin -u`
- mount `chromiumos_test_image.bin` manually
  - `losetup -P -f chromiumos_test_image.bin`
  - `mount -o ro /dev/loop0p3 sysroot`
  - `mount /dev/loop0p1 sysroot/mnt/stateful_partition`
  - `mount --bind sysroot/mnt/stateful_partition/dev_image sysroot/usr/local`

## USE flags

- `USE=angle -egl` and `arc-mesa-virgl`
  - with `USE=-egl`, the package disables egl/virgl and becomes vk-only
  - with `USE=angle`, the package
    - ships `init.gpu.rc` with `setprop ro.hardware.egl angle`
    - creates `/vendor/lib64/libGLESv2_angle.so` to point to
      `egl/libGLESv2_angle.so` provided by the vendor image
- `USE=cross_domain_context`
  - `vm_concierge` will start arcvm with `--gpu vulkan=true,context-types=cross-domain:...`

## DUT Misc Tips

- Chrome Tricks
  - `chrome://restart` in URL bar
- console login as `root/test0000`
- ssh login key is at `chromite/ssh_keys/testing_rsa`
- Upstart
  - configs are at `/etc/init`
  - initctl
    - reload
    - restart
    - start
    - stop
    - list
    - status
  - job `ui` starts `session_manager` which starts chrome processes
- Logs are under `/var/log`
- Disk layout
  - <https://chromium.googlesource.com/chromiumos/platform/crosutils/+/HEAD/build_library/disk_layout_v3.json>
  - sda1: users, data, 111G
  - sda2: kernel, 32M
  - sda3: rootfs, 4G
    - remount as rw: mount -o remount,rw /
  - sda{4,5}: second kernel/rootfs
  - sda8: OEM data, 16M (not used yet)
  - sda12: EFI, 32M
- Daemons
  - `init`, upstart
  - `udevd`
  - `frecon`, VT over KMS, boot animation
  - `agetty`
  - `auditd`
  - `rsyslogd`
  - `dbus-daemon`
  - `spaced`, disk space info
  - `mojo_service_manager`
  - `iioservice`
  - `btmanagerd`
  - `bluetoothd`
  - `daisydog`
  - `wpa_supplicant`
  - `hermes`
  - `powerd`, userspace power manager
  - `shill`, network connection manager
  - `pca_agentd`
  - `trunksd`, Trunks TPM Library
  - `tpm_managerd`
  - `chapsd`, PKCS#11
  - `bootlockboxd`
  - `attestationd`
  - `vtpmd`
  - `cryptohomed`, mounting encrypted home dirs
  - `secagentd`
  - `featured`
  - `secanomalyd`
  - `patchpaneld`
  - `permission_broker`
  - `dnsproxyd`
  - `mtpd`, Media Transfer Protocol (phones)
  - `cras`, Chromium OS Audio Server
  - `cros_healthd`
  - `ModemManager`
  - `dlcservice`
  - `dhcpcd`
  - `sshd`
  - `cros-disks`, mounting removable media
  - `avahi-daemon`
  - `dnsproxyd`
  - `sslh-fork`
  - `conntrackd`, netfilter conntrack
  - `update_engine`, runtime system update
    - <https://android.googlesource.com/platform/system/update_engine>
  - `upstart-socket-bridge`
  - `periodic_scheduler`
  - `temp_logger.sh`
  - `anomaly_detector`, crash reporter
  - `typecd`
  - `missived`
  - `u2fd`
  - `resourced`
  - `traced`
  - `traced_probes`
  - `cros_camera_algo`
  - `timberslide`, EC crash reporter
  - `debugd`
  - `fwupd`
  - `ml_service`
  - `tlsdated`, NTP
  - `metrics_daemon`, metric collection
  - `memd`, collect high memory pressure data
  - `cros_camera_service`, camera
  - `coreutils`
- after `start ui`
  - `session_manager`, starts chrome
  - `cdm-oemcrypto`
  - `private_computingd`
  - multiple chrome processes
    - one browser (no --type)
      - UI, manage tabs
    - three `--type=zygote`
    - two `--type=utility`
    - one `--type=gpu-process`
    - one `--type=-broker`
    - one `--type=renderer`
      - each tab is rendered by a renderer process

## Debug Buttons

- <https://chromium.googlesource.com/chromiumos/docs/+/main/debug_buttons.md>
  - on prototypes, might require `dut-control power_state:rec` to enter the
    recovery mode
- to reset EC, press `Refresh` and then `Power`
