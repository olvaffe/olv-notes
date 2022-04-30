GDM
===

## Overview

- <https://gitlab.gnome.org/GNOME/gdm>
- see `daemon/INTERNALS` for internal working
- gdm
  - it has objects of these classes
    - `GdmManager`
    - `GdmLocalDisplayFactory`
    - `GdmDisplayStore`
    - `GdmDisplay`
    - `GdmSlaveProxy`
- `gdm-simple-greeter`
  - it has objects of these classes
    - `GdmGreeterClient`
    - `GdmGreeterSession`

## gdm-simple-slave

- it has objects of these classes
  - `GdmSimpleSlave`
  - `GdmSlave`
  - `GdmServer`
  - `GdmGreeterServer`
  - `GdmWelcomeSession`
  - `GdmSessionDirect`
- is started by gdm
- in `gdm_simple_slave_run()`, it calls `gdm_server_start()` to start Xorg
- when Xorg is ready, it calls `gdm_slave_connect_to_x11_display()` to open
  the display
- then `start_greeter()` to start the greeter
  - it runs `/etc/gdm/Init/*` first
  - it creates `GdmGreeterServer`, which is a dbus server; creates
    `GdmGreeterSession`, which is a `GdmWelcomeSession`.  The session is run
    by spawning gnome-session, which eventually runs `gdm-simple-greeter`
- `create_new_session` creates a `GdmSessionDirect`
- `gdm-simple-greeter` connects to `GdmGreeterServer`, which starts a
  conversation
- All xsessions from these directories are listed
  - `/usr/share/gdm/BuiltInSessions`
  - `/usr/share/xsessions`
  - In `on_session_started`, `PreSession/Default` is run (in the session process)
- `gdm-simple-slave` spawns `gdm-session-worker` for PAM auth
- when a user is chosen and the password is entered, `gdm-session-worker`
  authenticates the credential, and notifies `gdm-simple-slave`
- finally, the greeter is stopped and `/etc/gdm/PostLogin/*` is run

## gdm-session-worker

- it has objects of these classes
  - `GdmSessionWorker`
  - `GdmSessionSettings`
    - `ActUserManager` from `AccountsService`
- it connects to
  - `/org/gnome/DisplayManager/Session`, `org.gnome.DisplayManager.Session`
    provided by `gdm-simple-slave`
- the worker responds to signals from the session server
  - after the greeter has the username, the worker receives `SetupForUser`.
    It uses the info to initialize PAM and calls `SetupComplete`.
  - it then receives `Authenticate`.  the worker asks PAM to authenticate
    the user, which results in `SecretInfoQuery` being called back.  After
    the user has been successfully authenticated, it calls `Authenticated`
  - the worker then receives `Authorize`, upon which it calls
    `pam_acct_mgmt()` and calls back with `Authorized`
  - Similar for `EstablishCredentials`, which results in worker calling
    `setgid()`, `pam_setcred()` and `Accredited`
  - Next, `OpenSession`.  It results in `pam_open_session()` and
    `SessionOpened`
  - Next, multiple `SetEnvironmentVariable` and `pam_putenv()`
  - Finally, `StartProgram`.  For the default session, the program is
    `/etc/gdm3/Xsession "default"`.  The worker calls
    `gdm_session_worker_start_session()` to fork, setuid, and exec the
    program.  Then the parent calls `SessionStarted` back.

  - The worker remains alive until the session ends.
