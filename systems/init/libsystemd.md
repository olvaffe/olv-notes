libsystemd
==========

## Overview

- `sd-daemon.h` provides apis to write a modern-style daemon
  - `sd_booted`
  - `sd_notify*` and `sd_pid_notify*`
  - `sd_listen_fds*`
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
