Chrome OS Development
=====================

## New Device

- enable developer mode
  - hold ESC and F3/Refresh, then press power button to boot into recovery
    mode
  - while in recovery mode, press Ctrl-D to enter developer mode
- select boot device
  - at developer mode warning, press Ctrl-D or Ctrl-U to boot from disk or usb
    - for USB boot, need to run `enable_dev_usb_boot` from console first
- console
  - while in developer mode, Ctrl-Alt-F2 to enter console
- rw rootfs
  - `/usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification`
  - `reboot`
- sshd
  - `/usr/libexec/debugd/helpers/dev_features_ssh`
  - `passwd`
- flash latest test image
  - `cros flash ${DUT_IP} xbuddy://remote/${BOARD}/latest-canary/test`
- confirm versions
  - H1 firmware: `gsctool -a -f`
  - EC firmware: `ectool version`
  - AP firmware: `crossystem fwid`
  - OS image: `grep CHROMEOS_RELEASE_DESCRIPTION /etc/lsb-release`

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
  - `build_image`
    - it builds images under ~/trunc/src/build/$BOARD
    - it always wipes the build directory clean, install packages, and done

## Portage

- man 5 portage
  - /etc/make.conf and /etc/portage/make.conf
  - PORTDIR="/usr/local/portage/stable" for path to the main repository
  - `PORTDIR_OVERLAY` for additional repositories
  - DISTDIR="/var/lib/portage/distfiles" for path to store downloaded sources
  - `PORT_LOGDIR="/var/log/portage` for log files
  - PKGDIR="/var/lib/portage/pkgs" where to store built packages
- check out board-specific /build/$BOARD/etc/make.conf
- an overlay is an ebuild repo
  - the main repo specified by PORTDIR is also an overlay
  - metadata/layout.conf
    - masters specifies other repos that this overlay can use and depend on
  - profiles/base/make.defaults
    - make.conf-like configs for the overlay
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

## Kernel

- "cros_workon --board=${BOARD} start chromeos-kernel-4_19"
- make changes to src/third_party/kernel/v4.19
  - or fetch a WIP branch, "git fetch cros <branch>"
    - https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/<branch>
- to build, "USE=\"kgdb vtconsole\" emerge-${BOARD} --nodeps chromeos-kernel-4_19"
- to deploy, "./update_kernel.sh --remote=<DUT IP address>"
- change kernel config
  - chromeos/config/*/*.config
- update kernel cmdline
  - on DUT: /usr/share/vboot/bin/make_dev_ssd.sh --edit_config
  - update_kernel.sh: edit src/build/images/<BOARD>/latest/config.txt

## Mesa

- "cros_workon --board=$BOARD start libdrm"
- make changes to src/third_party/libdrm
- "emerge-$BOARD libdrm"
- "cros deploy <DUT-IP> libdrm"
- same for mesa
  - "cros_workon --board=$BOARD start mesa"
  - make changes to src/third_party/mesa
  - "cros deploy <DUT-IP> mesa"

## Cross Compile

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
