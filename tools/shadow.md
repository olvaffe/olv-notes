shadow
======

## Overview

- `shadow`, <https://github.com/shadow-maint/shadow>
  - `useradd`, `userdel`, `usermod`
  - `groupadd`, `groupdel`, `groupmod`
  - etc.

## useradd

- `/etc/passwd` has seven fields
  - login name
  - optional encrypted password
    - if empty, no password is required
    - if `x`, the password is stored in `/etc/shadow` instead
    - otherwise, the password is encrypted with `crypt`
      - if it is not a valid output of `crypt`, such as `*` or `!`, it is
        impossible to login with a password
  - numerical user id
  - numerical group id
  - description (e.g., real name or comment)
  - user home directory
    - `login` sets `$HOME` to the value and `chdir`s to it
  - optional user command interpreter
    - `login` assumes `/bin/sh` if empty
    - `login` sets `$SHELL` to the value and `exec`s it
- `/etc/shadow` has nine fields
  - login name
  - encrypted password
  - date of last password change
    - `0` means the password should be changed on next login
  - minimum password age
  - maximum password age
  - password warning period
  - password inactivity period
  - account expiration date
  - reserved field
- files/directories potentially modified/created by `useradd`
  - `/home/$USER`
  - `/var/mail/$USER`
  - `/etc/passwd` and `/etc/shadow`
  - `/etc/group` and `/etc/gshadow`
  - `/etc/subuid` and `/etc/subgid`
- commonly used options
  - `-b`/`-d` for base/home directory
  - `-F` for subuid/subgid for system account
  - `-G` for supplementary groups
  - `-m` for home creation
  - `-r` for system account
    - the uid will be between `SYS_UID_MIN` and `SYS_UID_MAX` defined in
      `/etc/login.defs`
    - no subuid/subgid unless `-F`
    - no home dir unles `-m`
  - `-s` for shell, default to `SHELL` defined in `/etc/default/useradd`
- system account
  - normal accounts and system accounts are no different to most tools
    except for those who choose to distinguish them
    - systemd uses `uid_is_system` to distinguish them
  - system accounts usually do not need a home directory and do not need a
    shell
    - `useradd -r -d /nonexistent -s /usr/sbin/nologin <name>`
    - if a home directory is desired, `/var/lib/<name>` is one option
  - rootless podman containers are a bit ambiguous
    - they should be run by system accounts
    - but they usually need home dirs and `XDG_RUNTIME_DIR`, where the latter
      is usually created by `pam_systemd` during normal login
    - `useradd -r -F -m <name>`
    - `loginctl enable-linger <name>` to start the user instance on boot
