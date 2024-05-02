systemd-logind
==============

## Overview

- systemd-logind manages user logins
  - it manages vt
  - it spawns agetty
  - it manages user logins and user sessions
  - it manages seats and devices
  - it handles power/lid keys
  - and more

## `pam_systemd`

- `man pam_systemd`
  - `XDG_SESSION_ID`
  - `XDG_RUNTIME_DIR`
  - `XDG_SESSION_TYPE`
  - `XDG_SESSION_CLASS`
  - `XDG_SESSION_DESKTOP`
  - `XDG_SEAT`
  - `XDG_VTNR`

## `loginctl`

- show/control sessions, users, and seats
- `loginctl seat-status seat0`
  - shows user sessions and devices assigned to this seat
  - `/usr/lib/udev/rules.d/71-seat.rules` decides which devices belong to a
    seat
- `loginctl seat-status $UID`
  - shows the user slice
- `loginctl session-status <session>`
  - shows the user session
- `loginctl enable-linger <user>`
  - it calls `org.freedesktop.login1.Manager.SetUserLinger`
  - `method_set_user_linger` touches `/var/lib/systemd/linger/<user>` and
    calls `manager_add_user_by_uid`
  - when logind starts, `manager_enumerate_linger_users` scans the directory
    and calls `manager_add_user_by_name`

## Manager

- `Manager` is logind itself and is a singleton
  - it manages a list of `Device`s
  - it manages a list of `Seat`s
  - it manages a list of `Session`s
  - it manages a list of `User`s
  - it manages a list of `Inhibitor`s
  - it manages a list of `Button`s

## Device

- a `Device` is created for udev device that belongs to a seat
  - udev `71-seat.rules` tags `seat` to `sound`, `input`, `drm`, and other
    devices 
  - it also tags `master-of-seat` to `drm`
- `manager_dispatch_device_udev` is a monitor that is called for each udev
  device tagged with `seat`
  - it creates a `Device` and adds it to a `Seat`
  - if `Seat` is missing, it skips the device
- `manager_dispatch_seat_udev` is a monitor that is called for each udev
  device tagged with `master-of-seat`
  - it creates a `Device` and adds it to a `Seat`
  - if `Seat` is missing, it creates one on-demand

## Seat

- a `Seat` is a physical seat
  - there is always `seat0`
  - a seat has a list of attached `Device`s
  - a seat also has a list of `Session`s and the concept of the active session
    on the seat

## Button

- a `Button` is a power-switch button
  - `manager_enumerate_buttons` enumerates all input devices that are tagged
    `power-switch` and wrap them in `Button`s
- a `Button` opens and polls its input device; on events, `button_dispatch`
  is called to handle events such as shutdown, restart, suspend, etc.

## User

- a `User` is a logged-in user
  - it has a list of `Sesseion`s

## VT management

- it looks like
  - seat0 owns all VTs
  - VTs are put in `VT_PROCESS` mode
  - a session may belong to a VT
  - on VT switch, seat0 makes one of the sessions in the VT active and pauses
    the previous session
- `manager_connect_console`
  - opens and polls `/sys/class/tty/tty0/active`
    - the path returns the active VT (e.g., `tty3`) on read()
    - on VT switch, `manager_dispatch_console` is called.  It then calls
      `seat_read_active_vt` to update the active vt and active session for
      seat0

## AutoVT

- logind manages VTs
  - by putting them in `VT_PROCESS` mode, logind is in charge of VT switches
- when switching to a new VT, `manager_spawn_autovt` is called
  - it asks systemd to start `autovt@ttyN.service`
  - which is an alias of `agetty@.service`
  - which executes `-/sbin/agetty -o '-p -- \\u' --noclear %I $TERM`
    - `%I` expands to `ttyN`
    - `$TERM` expands to `linux`
- `agetty` prompts for user name
- `login` logs in the user

## User Session

- after a user logs in, `pam_systemd` together with `systemd-logind` does
  - starts `user-$UID.slice`.  Under which,
    - starts `user@$UID.service`
    - starts `session-$SESSION.scope`
- when $UID first logs in, `pam_systemd` starts `user@$UID.service`
  - the service depends on `user-runtime-dir@.service` which mounts tmpfs at
    `/run/user/$UID`
  - it then starts `systemd --user` under `user-$UID.slice`
- for each $UID login, `pam_systemd` also starts a session scope,
  `session-$SID.scope`
  - `login` process itself is moved to the scope
  - all future processes are also in this scope

## suspend

- systemd-logind opens evdev devices for
  - `KEY_POWER`/`KEY_POWER2`: power key
  - `KEY_SLEEP`: suspend key
  - `KEY_SUSPEND`: hibernate key
  - `SW_LID`: lid opened/closed
  - `SW_DOCK`: machine docked/undocked
- when lid closed, `button_lid_switch_handle_action` is called
  - enumerate DRM connectors in sysfs to see if any external display connected
  - enumerate `/sys/class/power_supply` to see if AC is online
  - depending on whether there are external display or whether AC is online,
    different actions can be taken
    - such as ignore, poweroff, reboot, halt, suspend, lock, etc.
  - by default,
    - when external displays are connected, lid closed is ignored
    - otherwise, suspend
