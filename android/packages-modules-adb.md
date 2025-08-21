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

## Transports

- USB
  - we want the dut to be in device mode rather than host mode
    - DRD can switch between modes
    - OTG is in device mode by default (and can switch to host mode)
    - host-to-host bridge cable allows a host to treat the other as a device
    - DbC requires a debug cable to advertise a device in device mode
      - <https://android.googlesource.com/platform/packages/modules/adb/+/d0db47dcdf941673f405e1095e6ffb5e565902e5>
  - local adb server runs `adb_server_main`
  - `usb_init` from `usb_linux.c` spawns a thread to scan `/dev/bus/usb`
  - it only considers usb devices with
    - device descriptor
      - `bLength                18`
      - `bDescriptorType         1`
    - config descriptor
      - `bLength                 9`
      - `bDescriptorType         2`
    - interface descriptor
      - `bLength                 9`
      - `bDescriptorType         4`
      - `bNumEndpoints           2`
      - `bInterfaceClass       255` (`ADB_CLASS`)
      - `bInterfaceSubClass     66` (`ADB_SUBCLASS`)
      - `bInterfaceProtocol      1` (`ADB_PROTOCOL`)
      - new adb also supports `ADB_DBC_CLASS` and `ADB_DBC_SUBCLASS`
    - endpoint descriptor
      - `bLength                 7`
      - `bDescriptorType         5`
      - `bmAttributes            2`
  - `register_device` registers a usb device
    - serial is from `serial` attr of the sysfs node
    - `register_usb_transport` calls `register_transport`
      - `send_connect` sends a connect packet to the device
        - after handshake, the device calls `send_connect` to send a connect back
      - `atransport::HandleRead` handles received packets
        - `handle_new_connection` handles connect packet
          - `parse_banner` changes the state from `offline` to `device`

## Adb Commands

- `adb devices`
  - this sends `devices` cmd to local adb server
  - `handle_host_request` calls `list_transports` to return `transport_list`
  - `register_transport` adds a transport to `transport_list`
- `adb connect`
  - this sends `connect` cmd to local adb server
  - `host_service_to_socket` creates a thread to run `connect_device`
  - `register_socket_transport` calls `register_transport`
- `adb push/pull/sync` creates a `SyncConnection` for file xfer
- `adb install/uninstall` runs `adb shell cmd package ...`
- `adb bugreport` runs `adb shell bugreportz -p` and saves the output to a
  local file
- `adb logcat` runs `adb shell logcat`
- `adb disable-verity/remount` runs `adb shell ...`
- `adb reboot fastboot` runs `reboot:fastboot` service
  - `reboot_device` runs `/system/bin/reboot fastboot`, which sets
    `ANDROID_RB_PROPERTY` (`sys.powerctl`) to `reboot,fastboot`
  - init `HandlePowerctlMessage` handles the property change
- `adb root` runs `root:` service
  - `restart_root_service` sets `service.adb.root` to 1

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

## Tips

- `adb shell am start -a com.android.setupwizard.FOUR_CORNER_EXIT` skips OOBE
