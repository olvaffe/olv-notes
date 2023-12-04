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
      - `/etc/pam.d/other`
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
- `display_login_messages` shows motd and check `MAIL` for new mails
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
