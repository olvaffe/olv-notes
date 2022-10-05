Chrome OS Overview
==================

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
- rw rootfs and ssh
  - see `/etc/init/openssh-server.conf.README`
  - select `Enable debugging features` in OOBE (out-of-box experience) will do
  - otherwise,
    - `/usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification`
    - `reboot`
    - `/usr/libexec/debugd/helpers/dev_features_ssh`
    - `passwd`
- flash latest test image
  - `cros flash ${DUT_IP} xbuddy://remote/${BOARD}/latest-canary/test`
- confirm versions
  - H1 firmware: `gsctool -a -f`
  - EC firmware: `ectool version`
  - AP firmware: `crossystem fwid`
  - OS image: `grep CHROMEOS_RELEASE_DESCRIPTION /etc/lsb-release`
  - `chromeos-firmwareupdate -t` to flash

# Host

## Source Code

- Follow Chromium OS Developer Guide to get the source code
  - git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
  - ./depot_tools/repo init -u https://chromium.googlesource.com/chromiumos/manifest.git
         --repo-url https://chromium.googlesource.com/external/repo.git
  - ./depot_tools/repo sync -q -j4
- Source Tree Layout
  - chromite, build tools and scripts
  - chromium, random code from Chromium
  - docs
  - infra, CI, test infrastructure
  - infra_virtualenv, python virtualenv that infra depends on
  - manifest*, repo manifests
  - src
    - aosp, random code from AOSP
    - overlays
    - platform
    - platform2
    - repohooks
    - scripts, to build and install packages in chroot
    - third_party, tons of third-party projects
    - weave, google weave
- Build Artifacts
  - chroot
  - devserver, images for use by dev server
  - src/build
- Portage
  - src/third_party/portage-stable, mirror Gentoo portage-stable?
  - src/third_party/chromiumos-overlay, ebuilds for Chromium OS-specific apps
  - overlays add board-specific ebuilds

## chroot

- minimal ChromiumOS distribution
  - "cat /etc/lsb-release" gives "DISTRIB_CODENAME=ChromiumOS"
  - is based on Gentoo
  - packages are installed by emerge
  - target packages (also ChromiumOS distributions) are managed by emerge-$BOARD
    - ./setup_board installs emerge-$BOARD and other binaries to /usr/local/bin
- /mnt/host/source, host repo sync root
- /opt, toolchains to build coreboot, Java VM, container/VM images, etc.
- /packages, cached binary emerge packages
- /usr/local/portage, portage-stable and chromiumos-overlay
- /var/cache/chromeos-cache/distfiles, emerge artifacts
- /build/$BOARD, emerge-$BOARD builds and installs packages here

# DUT

## firmwares

- see <bootloader/cros.md>

## Chromium OS

- Keyboard shortcuts
  - enter console: Ctrl-Alt-{F2,F3,...}
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
