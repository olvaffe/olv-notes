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
