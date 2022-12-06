Chrome OS Overview
==================

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
    - Ctrl-U to too from USB
    - `chromeos-install`
- write down firmware versions
  - H1 firmware: `gsctool -a -f`
  - EC firmware: `ectool version`
  - AP firmware: `crossystem fwid`
  - OS image: `grep CHROMEOS_RELEASE_DESCRIPTION /etc/lsb-release`
- follow <../../firmwares/cros.md> to update firmwares
- after updating firmwares, we might lose developer mode
  - `crossystem devsw_boot` should return 1
  - if not, re-enter the developer mode

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
    - also `~/chromiumos/chromite/scripts/gsutil config` and use
      `chromeos-bot` as the project id
  - `repo sync -j4`
- Source Tree Layout
  - chromite, build tools and scripts
  - chromium, random code from Chromium
  - docs
  - infra, CI, test infrastructure
  - `infra_virtualenv`, python virtualenv that infra depends on
  - `manifest*`, repo manifests
  - src
    - aosp, random code from AOSP
    - overlays
    - platform
    - platform2
    - repohooks
    - scripts, to build and install packages in chroot
    - `third_party`, tons of third-party projects
    - weave, google weave
- Portage
  - `src/third_party/portage-stable`, ebuilds copied from Gentoo portage-stable
  - `src/third_party/chromiumos-overlay`, ebuilds for Chromium OS-specific apps
  - overlays add board-specific ebuilds
- Build Artifacts
  - chroot
  - devserver, images for use by dev server
  - src/build

## SDK chroot

- `cros_sdk` will download and enter SDK chroot
- to build an image,
  - `./build_packages --board=BOARD --nowithdebug --internal`
  - `./build_image --board=BOARD --no-enable-rootfs-verification test`
- SDK is a cros (gentoo) chroot tarball
  - <https://chromium.googlesource.com/website/+/HEAD/site/chromium-os/build/sdk-creation/index.md>
  - `cros_sdk` downloads the tarball from, for example,
    <https://storage.googleapis.com/chromiumos-sdk/cros-sdk-2022.12.03.124835.tar.xz>
  - it untars the tarball into `~/chromiumos/chroot`
  - conceptually, to build a SDK tarball,
    - `./build_packages --board=amd64-host target-sdk`
- because SDK is a gentoo chroot, it can be updated without re-downloading the
  tarball
  - packages can be updated or installed by `emerge`
  - `./update_chroot` updates `target-sdk` package
- /mnt/host/source, host repo sync root
- /opt, toolchains to build coreboot, Java VM, container/VM images, etc.
- /packages, cached binary emerge packages
- /usr/local/portage, portage-stable and chromiumos-overlay
- /var/cache/chromeos-cache/distfiles, emerge artifacts
- /build/$BOARD, emerge-$BOARD builds and installs packages here

# DUT

## Chromium OS

- Chrome Tricks
  - chrome://restart in URL bar
- console login as root/test0000
- Upstart
  - configs are at "/etc/init"
  - initctl
    - reload
    - restart
    - start
    - stop
    - list
    - status
  - job "ui" starts "session_manager" which starts chrome processes
- Logs are under /var/log
- Disk layout
  - sda1: users, data, 111G
  - sda2: kernel, 32M
  - sda3: rootfs, 4G
    - remount as rw: mount -o remount,rw /
  - sda{4,5}: second kernel/rootfs
  - sda8: OEM data, 16M (not used yet)
  - sda12: EFI, 32M
- Daemons
  - third party
    - udevd, agetty, dbus-daemon, rsyslogd, wpa_supplicant, ModemManager
    - avahi-daemon, bluetoothd
    - conntrackd, netfilter conntrack
    - upstart (init)
    - tlsdated, NTP-like
      - https://github.com/ioerror/tlsdate
  - chromium os platform2
    - https://chromium.googlesource.com/chromiumos/platform2
    - powerd, userspace power manager
    - sessnion_manager, starts chrome
    - debugd
    - shill, network connection manager
    - midis, MIDI the audio
    - permission_broker, access /dev
    - mtpd, Media Transfer Protocol (phones)
    - cros-disks, mounting removable media
    - btdispatch, bluetooth
    - anomaly_collector, crash reporter
    - timberslide, EC crash reporter
    - trunksd, Trunks TPM Library
    - tpm_managerd
    - chapsd, PKCS#11
    - attestationd, certificates
    - cryptohomed, mounting encrypted home dirs
    - metrics_daemon, metric collection
    - memd, collect high memory pressure data
  - chromium os others
    - frecon, terminal emulator over KMS to replace VT
      - https://chromium.googlesource.com/chromiumos/platform/frecon/+/HEAD/DESIGN-DOC.md
    - cros_camera_service, camera
      - https://chromium.googlesource.com/chromiumos/platform/arc-camera/
    - cras, Chromium OS Audio Server
      - https://chromium.googlesource.com/chromiumos/third_party/adhd/
    - minijail, put programs in jail
      - https://android.googlesource.com/platform/external/minijail/
    - update_engine, runtime system update
      - https://android.googlesource.com/platform/system/update_engine/
  - android
  - chrome
    - one browser (no --type)
      - UI
      - manage tabs
    - some --type=renderer
      - each tab is rendered by a renderer process
    - two --type=zygote
    - one --type=gpu-process
    - one --type=-broker

## SSH

- Login with testing ssh key
  - "ssh -i ~/chromiumos/chromite/ssh_keys/testing_rsa root@<DUT-IP>"
- ssh_config
  - User root
  - IdentityFile ~/chromiumos/chromite/ssh_keys/testing_rsa
  - ControlMaster auto
  - ControlPath ~/.ssh/control-%r@%h:%p
  - ControlPersist yes
  - LocalForward 1234 localhost:1234 # for gdbserver

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
