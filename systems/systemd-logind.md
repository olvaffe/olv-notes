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

## seatd

- <https://git.sr.ht/~kennylevinsen/seatd>
  - the codebase is smaller
- initialization
  - `server_init` calls `seat_create` with `seat0` and `vt_bound` set
  - `open_socket` listens on `/run/seatd.sock`
  - `poller_poll` polls the socket
- `server_handle_connection` handles connections to the socket
  - `server_add_client` calls `client_create` to create a client
- `client_handle_connection` handles client opcodes
  - `CLIENT_OPEN_SEAT` is handled by `handle_open_seat`
    - `seat_add_client`
      - `seat_update_vt` sets `seat->cur_vt` and `client->session` to the
        current vt (usually 1)
    - `seat_open_client`
      - `vt_open` opens the current vt and switches to
        `VT_PROCESS`/`KD_GRAPHICS`/etc
  - `CLIENT_CLOSE_SEAT` is handled by `handle_close_seat`
    - `seat_remove_client` closes all opened devices
    - `seat_close_client` closes the current client (and activates the next
      client if any)
      - `vt_close` restores the vt
  - `CLIENT_OPEN_DEVICE` is handled by `handle_open_device`
    - `seat_open_device` opens a device node and adds it to `client->devices`
      - if `SEAT_DEVICE_TYPE_DRM`, the fd is made drm master
      - if `SEAT_DEVICE_TYPE_EVDEV`, no special handling
  - `CLIENT_CLOSE_DEVICE` is handled by `handle_close_device`
    - `seat_close_device` closes the device
  - `CLIENT_SWITCH_SESSION` is handled by `handle_switch_session`
    - because `vt_bound` is set, `vt_switch` initiates the switch 
  - `CLIENT_DISABLE_SEAT` is handled by `handle_disable_seat`
    - `seat_ack_disable_client` acks the disabling of the active client
  - `CLIENT_PING`
- session switch
  - user presses `CTRL-ALT-Fx`
  - compositor receives `XKB_KEY_XF86Switch_VT_x` and calls
    `libseat_switch_session` 
  - seatd receives `CLIENT_SWITCH_SESSION`
    - `vt_switch` switches vt
    - `server_handle_vt_rel` calls `seat_vt_release`
      - `seat_disable_client` disables the active client and sends
        `SERVER_DISABLE_SEAT` to the client
      - compositor calls `libseat_disable_seat` in response to
        `SERVER_DISABLE_SEAT`
        - seatd receives `CLIENT_DISABLE_SEAT`
      - `vt_ack` releases the vt
    - `server_handle_vt_acq` calls `seat_vt_activate`
      - `seat->cur_vt` is updated
      - `vt_ack` acquires the vk
      - `seat_activate` picks the next client for the new vt and calls
        `seat_open_client` to open the client and send `SERVER_ENABLE_SEAT`

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

## agetty

- logind manages VTs
  - by putting them in `VT_PROCESS` mode, logind is in charge of VT switches
- when switching to a new VT, `manager_spawn_autovt` is called
  - it asks systemd to start `autovt@ttyN.service`
  - which is an alias of `agetty@.service`
  - which executes `-/sbin/agetty -o '-p -- \\u' --noclear %I $TERM`
    - `%I` expands to `ttyN`
    - `$TERM` expands to `linux`
- agetty
  - runs as root
  - opens `/dev/ttyN`
  - calls `tcgetsid` and `TIOCSCTTY` to make sure `/dev/ttyN` is the
    controlling terminal
  - closes `STDIN_FILENO` and reopens `/dev/ttyN` to make sure it is stdin
  - calls `tcsetpgrp` to make itself foreground
  - prompts `login: ` and reads login name
  - `execv`s `login` wit the login name
- login
  - runs as root
  - closes stdin/stdout/stderr
  - calls `vhangup`
  - reopens `/dev/ttyN` for stdin/stdout/stderr
  - initializes pam
  - calls `pam_authenticate` to authenticate the username
  - calls `initgroups` to init the supplementary groups
  - opens pam session
  - calls `setgid`
  - sets up environ (`TERM`, `HOME`, `USER`, `SHELL`, etc.)
    - `TERM` is from `agetty`
    - the others are from `/etc/passwd`
  - displays motd
  - calls `TIOCNOTTY` to detach the controlling tty
  - forks
    - the parent
      - closes all fds
      - calls `wait` repeatedly until there is no child
      - closes pam session
    - the child
      - calls `setsid` to create a new session and becomes the leader
      - reopens `/dev/ttyN` for stdin/stdout/stderr
      - calls `TIOCSCTTY` to attach the controlling tty
      - calls `setuid`
      - calls `chdir` to home
      - calls `execvp` to execute the shell
- environment variables
  - kernel invokes `init` with `HOME=/` and `TERM=linux`
  - systemd has
    - `TERM=linux`
    - added by initramfs shell
      - `PWD=/`
      - `SHLVL=1`
    - added by initramfs `/init`
      - `PATH=/sbin:/usr/sbin:/bin:/usr/bin`
      - `drop_caps=`
      - `init=/sbin/init`
      - `rootmnt=/root`
  - systemd-logind has ...
  - agetty has ...
  - login has ...
  - bash has ...

## User Login

- agetty prompts for user name
- after getting the user name, agetty `exec`s login
- login prompts for user password and authenticates using PAM
- `/etc/pam.d/login` includes `/etc//pam.d/system-login`.  Among other things,
  it include `pam_systemd`
- `pam_systemd` together with `systemd-logind` does
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
