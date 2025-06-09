libsystemd
==========

## Overview

- `sd-daemon.h` provides apis to write a modern-style daemon
  - `sd_booted`
  - `sd_notify*` and `sd_pid_notify*`
    - this sends message to `NOTIFY_SOCKET`
  - `sd_listen_fds*`
    - this listens to `LISTEN_FDS`
  - `sd_watchdog_enabled`
  - `sd_is_*`
- `sd-journal.h` provides apis to talk to `systemd-journald`
  - `sd_journal_*`
- `sd-login.h` provides apis to talk to `systemd-logind`
  - `sd_get_*`
  - `sd_session_get_*` `sd_session_is_*`
  - `sd_uid_get_*` `sd_uid_is_*`
  - `sd_seat_get_*` `sd_seat_can_*`
  - `sd_pid_get_*` and `sd_pidfd_get_*`
  - `sd_machine_get_*`
  - `sd_peer_get_*`
  - `sd_login_monitor_*`
- `sd-device.h` provides apis to talk to `systemd-udev`
  - it deprecates `libudev.h`
  - `sd_device_*`
- `sd-bus.h` provides apis to write a dbus client
  - `sd_bus_*`
- `sd-event.h` provides apis to write an `epoll`-based event loop
  - `sd_event_*`
- `sd-hwdb.h` provides direct read-only access to hwdb
  - `sd_hwdb_*`
- `sd-id128.h` provides apis to work with 128-bit ids
  - `sd_id128_*`
- `sd-path.h` provides apis to look up xdg paths
  - `sd_path_lookup*`

## `sd-device`

- `sd_device` works with a device
  - `sd_device_new_from_*` creates an `sd_device`
    - `sd_device_new_from_syspath` creates from a `/sys/...` dir
      - the dir should contain `uevent`
    - `device->syspath` is set to the sysfs dir of the device
    - `device->properties` has one property
      - `DEVPATH`, which is the sysfs path without `/sys` prefix
  - `sd_device_get_property_value` gets a prop val
    - `device_read_uevent_file` parses sysfs `uevent` on demand
      - props are added to `device->properties`
      - some are also set to `device` fields for quick access
    - `device_read_db` parses hwdb for the device on demand
      - it parses `/run/udev/data/<device-id>`
      - `E:` lines are props and are added to `device->properties`
- `sd_device_enumerator` enumerates devices
- `sd_device_monitor` monitors hotplugs
