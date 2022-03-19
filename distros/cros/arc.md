ARC
===

## `session_manager`

- after `session_manager` starts chrome, chrome requests "StartArcMiniContainer"
  - request handled by `session_manager` again
  - `session_manager` sends upstart "start-arc-instance" impulse
  - it also calls `run_oci start`, which starts an OCI container
  - it reads container /init pid from `/run/containers/android-run_oci`
- chrome requests "UpgradeArcContainer" after user logs in
  - `session_manager` sends upstart "continue-arc-boot" impulse
- once chrome considers ARC booted, it requests "EmitArcBooted"
  - `session_manager` sends upstart "arc-booted" impulse
- when "StopArcInstance" is requested (or when container /init crashed)
  - `session_manager` requests container /init to exit
  - after /init exits, it calls `run_oci destroy` to shutdown the container
  - it sends upstart "stop-arc-instance" impulse

## `run_oci` and arc-setup

- when `run_oci start` is called, these hooks are invoked
  - precreate
  - prechroot
  - prestart
  - <start container and run /init>
  - poststart
- when `run_oci destroy` is called, these hooks are invoked
  - poststop
- `run_oci` parses "<container-bundle>/config.json" for hooks
  - precreate maps to "arc-setup --mode=setup"
  - prechroot maps to "arc-setup --mode=pre-chroot"
- arc-setup parses "/usr/share/arc-setup/config.json" for configs
- arc-setup has these modes
  - triggered by `run_oci start`
    - setup, setup mountpoints and etc.
    - pre-chroot, more mountpoints and write some files
  - triggerd by `start-arc-instance`
    - read-ahead
  - triggerd by "continue-arc-boot"
    - boot-continue, set up /data parition, maybe start adbd,  run
      "/system/bin/arcbootcontinue" inside Android, start arc-sdcard, etc.
    - mount-sdcard, indirectly triggered to mount sdcard
  - triggerd by "arc-booted"
    - update-restorecon-last
  - triggerd by "stop-arc-instance"
    - stop

## Development

- Logs can be found at /var/log/arc.log
- "android-sh" gives you shell
  - it gets container pid and root from `/run/containers/android-run_oci/container.pid`
  - it executes "nsenter" to enter the name space
  - it uses "runcon" to start shell in the desired context

## adb

- enable adb in settings
  - or, `android-sh -c "start adbd"`
  - or, `android-sh -c "setprop persist.sys.usb.config adb"`
- `adb connect 100.115.92.2:5555`
- public key auth
  - `adb keygen adbkey`
  - `cat ~/.android/adbkey.pub | android-sh -c "cat > /data/misc/adb/adb_keys"`
  - `android-sh -c "restorecon /data/misc/adb/adb_keys"`
- DUT with test image runs `sslh` to redirect adb packets to DUT:22 to 100.115.92.2:5555
  - host can `adb connect <DUT-IP>:22`

## CTS

- https://source.android.com/compatibility/cts/downloads.html
- Download Android Studio to get adb and aapt
- PATH=$PATH:~/Android/Sdk/build-tools/28.0.3 ./cts-tradefed run commandAndExit cts \
    --skip-device-info -m CtsGraphicsTestCases -t <CLASS>#<METHOD>

## Cross-Compile for ARC++ P

- `FEATURES="noclean" emerge arc-foo` and find the generated meson cross files
  from under `/build/$board/tmp/portage` as usual
  - requires `SYSROOT=/build/$board` as usual
  - the pkg-config wrapper additionally requires `ARC_SYSROOT` and `ABI`
- links
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/sys-devel/arc-toolchain-p/>
    - tarball has toolchains and sysroots for x86 and arm, both 32-bit and 64-bit
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/sys-devel/arc-build/>
    - .pc files for use with mesa
  - <https://mesonbuild.com/Cross-compilation.html>
