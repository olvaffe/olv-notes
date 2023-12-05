login and PAM
=============

## login

- repos
  - <https://github.com/util-linux/util-linux>
  - <https://github.com/linux-pam/linux-pam>
- `/etc/login.defs` is the config file
  - it is shared by multiple tools and only a subset of configs are applicable
    to login
- agetty initializes tty and prompts for user name
  - it runs as root
  - it opens `/dev/ttyN`
  - it calls `tcgetsid` and `TIOCSCTTY` to make sure `/dev/ttyN` is the
    controlling terminal
  - it closes `STDIN_FILENO` and reopens `/dev/ttyN` to make sure it is stdin
  - it calls `tcsetpgrp` to make itself foreground
  - it prompts `<hostname> login: ` and reads login name
  - it `execv`s `login -- <username>`
- `init_tty`
  - it gets the terminal path, `/dev/ttyN`, via `ttyname(STDIN_FILENO)`
  - it opens the termianl and dups the fd to stdin/stdout/stderr
- `init_loginpam`
  - `pam_start("login", username, &conv, &pamh);`
    - `username` is specified by `agetty`
    - `conv` is `misc_conv` from `libpam_misc`
    - this function allocates a `pam_handle_t` and reads both
      - `/etc/pam.d/<service>`
        - these rules are added to `pamh->handlers.conf`
      - `/etc/pam.d/other`
        - these rules are added to `pamh->handlers.other`, which is used only
          when `pamh->handlers.conf` is empty
    - it `dlopen()` the necessary modules
  - it passes information for PAM modules
    - `pam_set_item(pamh, PAM_RHOST, NULL);`
    - `pam_set_item(pamh, PAM_TTY, tty);`
      - tty is from `ttyname(0)` so is normally `/dev/ttyN`
    - `pam_set_item(pamh, PAM_USER_PROMPT, "<hostname> login:");`
- `loginpam_auth`
  - auth: `pam_authenticate (pamh, 0);`
    - it is called up to `LOGIN_MAX_TRIES` (3) times
    - if it fails for the first time, the user name is cleared with
      `pam_set_item(pamh, PAM_USER, NULL)` such that pam prompts for a new
      user name
    - when modules call `pam_get_user()`, and there is no `PAM_USER`, it
      starts a `PAM_PROMPT_ECHO_OFF` conversation with the app to get the user
      name using `PAM_USER_PROMPT` as the prompt.  The result is set to
      `PAM_USER` for later use.
    - modules start a `PAM_PROMPT_ECHO_OFF` conversation with the app to get
      the password as well.  For `pam_unix`, the prompt is localized
      `Password:`.  The password is set to `PAM_AUTHTOK` for later use.
    - the auth token for the user is then verified
- `loginpam_acct`
  - account: `pam_acct_mgmt (pamh, 0);`
    - it returns `PAM_SUCCESS` if the account is valid and active
    - if the account requires to update its password, `PAM_NEW_AUTHTOK_REQD` is
      returned
      - the app should call `pam_chauthtok(pamh, PAM_CHANGE_EXPIRED_AUTHTOK);`
        to update the password
  - the app retrieves the user name back from `PAM_USER`
- `xgetpwnam`
  - it calls `getpwnam_r` to look up the user from `/etc/passwd`
- it calls `initgroups` to init the supplementary groups
- `loginpam_session`
  - `pam_setcred(pamh, PAM_ESTABLISH_CRED);`
    - this calls into modules such that modules can retrieve the credential
      from the core
  - session: `pam_open_session(pamh, 0);`
    - to denote the start of a session
    - modules can be various things (before changing uid/gid)
  - `pam_setcred(pamh, PAM_REINITIALIZE_CRED);`
- logging
  - `log_utmp` updates `/var/run/utmp` and `/var/log/wtmp`
  - `log_audit` logs to audit
  - `log_lastlog` logs to `_PATH_LASTLOG` (`/var/log/lastlog`) and prints
    `Last login: <time>`
- `chown_tty` changes ownership of `/dev/ttyN`
- it calls `setgid` to change the gid
- `init_environ` initializes env
  - `HOME`, `USER`, are `SHELL` are from passwd
  - `TERM` is preserved
  - `PATH` is set to `_PATH_DEFPATH` (`/usr/local/bin:/usr/bin`)
  - `MAIL` is set to `_PATH_MAILDIR/<username>` (`/var/spool/mail`)
  - `LOGNAME` is set to `USER`
    - `su` may change `USER` and `LOGNAME` is never changed
  - env from pam is retrieved from `pam_getenvlist`
    - the app and the modules may call `pam_putenv()` to add/delete/modify
      environment variables attached to pam handle
    - this function dumps the attached env variables
- `display_login_messages` shows motd and check `MAIL` for new mails if
  enabled
- `fork_session`
  - it detaches the tty with `TIOCNOTTY`
  - the parent is kept alive with uid 0 and gid of the user.  It `wait`s
    for the child and calls
    - `pam_setcred(cxt->pamh, PAM_DELETE_CRED)`
    - `pam_end(cxt->pamh, pam_close_session(cxt->pamh, 0))`
  - the child calls
    - `setsid` to start a session
    - `open_tty` to open tty and dups it to stdin/stdout/stderr
    - `TIOCSCTTY` to make the tty the the controlling terminal
- the child goes on and...
  - `setuid` to set the uid
  - `chdir` to `HOME`
  - `pam_end` to end pam
  - `execvp` to invoke the shell
  - `execle(shell)`

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
