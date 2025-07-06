PAM
===

## Overview

- <https://github.com/linux-pam/linux-pam>

## Environment Variables

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
- agetty and login have
  - `TERM`
  - `PATH`
  - added by systemd
    - `SYSTEMD_EXEC_PID`
    - `INVOCATION_ID`
- user shell has
  - added by `login` using `/etc/passwd` and `/etc/login.defs`
    - `HOME`
    - `USER`
    - `SHELL`
    - `MAIL`
    - `LOGNAME`
  - added by `pam_env` using `/etc/environment`
  - added by `pam_systemd`
    - `XDG_SESSION_ID`
    - `XDG_RUNTIME_DIR`
    - `MAIL`
    - `LANG`
    - `XDG_SESSION_CLASS`
    - `XDG_SESSION_TYPE`
    - `XDG_SEAT`
    - `XDG_VTNR`
  - added by user services using `systemctl --user set-environment`
    - `DBUS_SESSION_BUS_ADDRESS`
  - added by shell using `/etc/profile`, etc.

## PAM config

- rule format
  - `service type control module-path module-arguments`
- `service` is omitted when in `/etc/pam.d/<service>`
- `type` is
  - `auth`
  - `account`
  - `password`
  - `session`
  - a `-` prefix means, when the module is missing, do not log to system log
- `control` is, in modern form, `[val1=act1...]` list
  - `val` matches the return value, such as
    - `success`
    - `default` matches all values not listed
  - `act` is
    - `ignore` means, if the module stack has more than one module, the
      return code does not affect the ultimate result
    - `bad` means the return value reflects the ultimate result
    - `die` means `bad` plus immediate termination
    - `ok` means, if there is no prior failure, the return value reflects
      the ultimate result
    - `done` means `ok` plus immediate termination
    - `N` skips the next N mouldes
    - `reset` means to reset the ultimate result
- `control` also has a simple form
  - `required` is `[success=ok new_authtok_reqd=ok ignore=ignore default=bad]`
  - `requisite` is `[success=ok new_authtok_reqd=ok ignore=ignore default=die]`
  - `sufficient` is `[success=done new_authtok_reqd=done default=ignore]`
  - `optional` is `[success=ok new_authtok_reqd=ok default=ignore]`
  - `include` means to include rules of the same type from another file
  - `substack` is unused
- `module-path` is absolute or relative to `/lib/security`
- `module-arguments` is a space-separated list of module-specific args
  - use `[]` to espace all space in an arg

## PAM modules

- on arch, `auth` is
  - `auth requisite                    pam_nologin.so`
    - `/var/run/nologin` and `/etc/nologin` must not exist
  - `auth required                     pam_shells.so`
    - user shell must be listed in `/etc/shells`
  - `auth requisite                    pam_nologin.so`
  - `auth required                     pam_faillock.so     preauth`
    - lock account for a period if fails consecutively for a few times
    - config file is `/etc/security/faillock.conf`
  - `-auth [success=2 default=ignore]  pam_systemd_home.so`
    - it replaces `pam_unix` if systemd-homed is used
  - `auth [success=1 default=bad]      pam_unix.so         try_first_pass nullok`
    - prompt for password and check against `/etc/shadow`
  - `auth [default=die]                pam_faillock.so     authfail`
  - `auth optional                     pam_permit.so`
    - always true
  - `auth required                     pam_env.so`
    - add/remove/modify envvars according to `/etc/security/pam_env.conf`
    - add envvars listed in `/etc/environment`
  - `auth required                     pam_faillock.so     authsucc`
- on arch, `account` is
  - `account required                    pam_access.so`
    - use `/etc/security/access.conf` for login access control
  - `account required                    pam_nologin.so`
  - `-account [success=1 default=ignore] pam_systemd_home.so`
  - `account required                    pam_unix.so`
    - it checks `/etc/shadow` for account/password expiracy
    - `man 5 shadow`
  - `account optional                    pam_permit.so`
  - `account required                    pam_time.so`
    - `/etc/security/time.conf` can lock out certain accounts on certain times
- on arch, `password` is
  - `-password [success=1 default=ignore] pam_systemd_home.so`
  - `password  required                   pam_unix.so try_first_pass nullok shadow`
    - this updates user's password (if `account` result is
      `PAM_NEW_AUTHTOK_REQD` and `pam_chauthtok` is called)
  - `password  optional                   pam_permit.so`
- on arch, `session` is
  - `session optional   pam_loginuid.so`
    - call `audit_setloginuid`
  - `session optional   pam_keyinit.so      force revoke`
    - create a user session keyring
  - `-session optional  pam_systemd_home.so`
  - `session required   pam_limits.so`
    - call `setrlimit` according to `/etc/security/limits.conf`
  - `session required   pam_unix.so`
    - log login/logout to syslog
  - `session optional   pam_permit.so`
  - `session optional   pam_motd.so`
    - display motd from various sources, such as `/etc/motd`, `/run/motd`,
      etc.
  - `session optional   pam_mail.so         dir=/var/spool/mail standard quiet`
    - check `/var/spool/mail/<user>` for new mails
  - `session optional   pam_umask.so`
    - apply umask using `/etc/login.defs`
    - if `usergroups`, which is default on debian, owner bits are set to group
      bits (`022` becomes `002`)
      - on systems where users share their primary groups, `022` ensures only
        users can modify their files by defaults
      - but when each user has their own primary group (aka user groups or
        user private groups), `002` is equally safe and is more convenient
  - `-session optional  pam_systemd.so`
    - register the user session to `systemd-logind` and thus the cgroup
      hierarchy
    - mount tmpfs to `/run/users/$UID`
    - set `XDG_SESSION_ID` to `/proc/self/sessionid` set by `pam_loginuid.so`
    - a user slice unit and a scope unit are created
    - `user@$UID.service` is started
    - umask, nice, and resource limits are applied
    - on logout of the last user session, undo everything
      - e.g., user slice, `user@$UID.service`, `/run/users/$UID`, etc.
  - `session required   pam_env.so`
