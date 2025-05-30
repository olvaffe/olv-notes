Android ADB
===========

## `/init`

- it listens to `/dev/socket/property_service`
  - `property_set(...)` sends a message to the socket
- The CLI tool `start <something>` is equivalient to
  `property_set("ctrl.start", "<service>");`
  - `/init` runs the service defined by the RC file
  - the caller must be system or root
- For other properties, the table `property_perms` defines the permissions
- Properties are stored in a shared memory
  - `persist.` properties will also be stored on the disk
- `property_get(...)` gets the property from the shared memory

## `adbd`

- `GUI -> Settings -> Applications -> Development -> Enable Debugging` sets
  `persist.service.adb.enable` to true
  - see `services/java/com/android/server/SystemServer.java`
  - it triggers the adbd service
- for non-engineering non-debugging build, `ro.secure` is set to 1 and
  `ro.debuggable` is set to 0
  - this puts `adbd` in secure mode.  `adbd` is run as user `shell`
  - `adbd` listens to port 5037 when not in secure mode
  - `adb root` will print `adbd cannot run as root in production builds`
- When there is `/dev/android_adb`, `adbd` opens it for connections.  Otherwise,
  it listens to port 5555

## Rooting

- Rooting is done by flashing a kernel with debuggable initramfs
  - `ro.debuggable=1` in `default.prop`
  - this makes `adb root` work

## internals

- adb can be run on host and device
  - it is called `adbd` when on the device
- It calls `usb_init`
  - on device, it opens `/dev/android_adb`, which waits for connection.
  - on host, it scans `/dev/bus/usb` for usb devices supporting adb
    - the device might be both readable and writable.
- It calls `local_init`
  - on device, it listens on 5555 port.
  - on host, it connects to 5555 port of `ADBHOST`.

## Remount

- <https://android.googlesource.com/platform/system/core/+/refs/heads/main/fs_mgr/README.overlayfs.md>
- `adb disable-verity` or `adb remount`
  - client translates it to `adb shell disable-verity` or `adb shell remount`
  - cmdline source code at `system/core/fs_mgr`
- to enable/disable verity,
  - `AVB_VBMETA_IMAGE_FLAGS_HASHTREE_DISABLED` in vbmeta partition is
    cleared/set
  - it requires a reboot, and if `-R`, is specified, it reboots automatically
    by setting `sys.powerctl` to `reboot,<progname>`
- to remount,
  - it disables verity first, and reboot automatically if `-R` is specified
  - it remounts overlayfs

## Command Tools

- `setprop` / `getprop`
  - set/get Android system properties
    - <https://source.android.com/docs/core/architecture/configuration/add-system-properties>
  - system properties are stored `/dev/__properties__`, a tmpfs
    - if a property starts with `persist.`, it is also written to
      `/data/property/persist.foo` on write
  - on boot, the tmpfs is populated using
    - `/default.prop`
    - `/system/build.prop`
    - more
    - `/data/property/*`
- `settings`
  - actions
    - `list`
    - `get`
    - `put`
    - `delete`
  - docs
    - <https://developer.android.com/reference/android/provider/Settings.Global>
    - <https://developer.android.com/reference/android/provider/Settings.System>
    - <https://developer.android.com/reference/android/provider/Settings.Secure>
  - internally, settings are stored in a sqlite database
    - `/data/data/com.android.providers.settings/databases/settings.db`
- `run-as`
  - <https://android.googlesource.com/platform/system/core.git/+/refs/heads/master/run-as/run-as.cpp>
- `appops`
- `pm`
  - `list packages`
  - `dump <package>`
  - `grant <package> <permission>`
    - `dump <package>` to see `requested permissions`
- `am`
  - `start-activity <INTENT>`
  - `broadcast <INTENT>`
  - `<INTENT>`
    - `-n <PACKAGE>/<ACTIVITY>`
    - `-e <EXTRA_KEY> <EXTRA_STR_VALUE>`
    - activity can access the extra strings by `getIntent().getStringExtra(name)`
  - `instrument -w -e <name> <value> <TEST_PACKAGE>/<RUNNER_CLASS>`
    - class's `onCreate` calls `bundle.getString(name)` to access name/value arguments
- `dumpsys`
