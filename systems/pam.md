PAM
===

## login, as an example

- `pam_start("login", NULL /* username */, &conv, &pamh);`
  - `conv` is `misc_conv` from `libpam_misc`
  - this function allocates a `pam_handle_t`.  It reads
    - `/etc/pam.d/<service>`
    - `/etc/pam.d/other`
    and `dlopen()` the necessary modules
- pass information for PAM modules
  - `pam_set_item(pamh, PAM_RHOST, "");`
  - `pam_set_item(pamh, PAM_TTY, tty);`
    - tty is from `ttyname(0)` so is normally `/dev/tty?`
  - `pam_fail_delay (pamh, 1000000 * delay);`
    - delay `delay` seconds on failure
  - `pam_set_item(pamh, PAM_USER_PROMPT, "<hostname> login:");`
  - `pam_set_item (pamh, PAM_USER, username);`
    - `username` is NULL if not specified from the command line
- auth: `pam_authenticate (pamh, 0);`
  - the modules will call `pam_get_user()` to get the user name.  Since it is
    usually not specified, PAM will start a `PAM_PROMPT_ECHO_OFF` conversation
    with the app to get it using `PAM_USER_PROMPT` as the prompt.  The result
    is set to `PAM_USER` for later use.
  - the modules will start a `PAM_PROMPT_ECHO_OFF` conversation with the app
    to get the password.  For `pam_unix`, the prompt is localized `Password:`.
    The password is set to `PAM_AUTHTOK` for later use.
  - the auth token for the user is then verified
- account: `pam_acct_mgmt (pamh, 0);`
  - it returns `PAM_SUCCESS` if the account is valid and active
  - if the account requires to update its password, `PAM_NEW_AUTHTOK_REQD` is
    returned
  - the app should call `pam_chauthtok(pamh, PAM_CHANGE_EXPIRED_AUTHTOK);` to
    update the password
- session: `pam_open_session(pamh, 0);`
  - to denote the start of a session
  - modules can be various things (before changing uid/gid)
- the app retrieves the user name back from `PAM_USER`, and use it to look up
  in `/etc/passwd`.  With the information, it
  - `setgid(gid)`
  - `initgroups(gid)`
- `pam_setcred(pamh, PAM_ESTABLISH_CRED);`
  - note, according to the man page, it should be called before
    `pam_open_session()`.  It should also be called with `PAM_DELETE_CRED`
    after `pam_close_session()`
- the app is ready to `fork()`
  - the parent is kept alive with uid 0 and gid of the user.  It waits
    for the child and call
    - `retcode = pam_close_session(pamh, 0);`
    - `pam_end(pamh, retcode);`
- the child does these, using the information from `/etc/passwd`
  - `update_utmp()`
  - `setuid(uid);`
  - `chdir(home);`
  - set these environment variables
    - HOME
    - SHELL
    - USER (current user)
    - LOGNAME (same as USER for login)
      - note after `su`, USER may change
    - PATH (as set in `/etc/login.defs`)
    - `pam_getenvlist(pamh)`
      - the app and the modules may call `pam_putenv()` to add/delete/modify
        environment variables attached to pam handle
      - this function dumps the attached env variables
    - `TERM`
  - `execle(shell)`
