Systemd Unit Configuration
==========================

## Units

- service units control daemons or command to be invoked
  - such as udevd or `udevadm trigger` (oneshot)
  - if `foo.service.wants/` exists, units in the directory are added as
    `Wanted=` dependency of the service unit
  - services needs to be activated by other units
- socket units activate services when a socket/fifi/netlink/etc has incoming
  traffic
  - e.g, `/run/udev/control`
  - it activates the server of the same name
- target units are used to group units, which can be used as synchronization
  points
  - `halt.service` is after `shutdown.target`, `umount.target`, and
    `final.target`
  - if `foo.target.wants/` exists, units in the directory are added as
    `Wanted=` dependency of the target unit
- device units encodes information about devices
  - they are dynamically generated for all udev devices tagged with
    `systemd`
  - if an udev device also has `SYSTEMD_WANTS` property, the device unit
    will depend on the wanted units
  - e.g., sound devices are tagged and want `sound.target`
- mount units describe mount points to be managed by systemd
  - `tmp.mount` can mount tmpfs on `/tmp`
- automount units describe mount points to be automounted
  - they must have mount units of the same name
- snapshot units are states of systemd at a point
  - there is not unit configuration files
  - snapshot units are created via `systemctl snapshot`
  - to return to a snapshot, run `systemctl isolate`
- timer units can be used for timer-based activation
  - for a timer unit, a matching unit file of another type must exist
  - `systemd-tmpfiles-clean.timer` runs `systemd-tmpfiles` every day (as
    well as 15 minutes after boot)
- swap units encode information about swap devices or files
  - they must be named after the devices they control
- path units define pathes to be monitored, enabling path-based activation
  - a matching service unit file must exist
- `man systemd.unit`
  - `Wants=`: if this unit gets started, other units listed are also started;
    it is fine for other units to fail
    - this does not express ordering; this unit and the wanted units may be
      started simultaneously unless `Before=` and `After=` are specified
  - `Requires=`: if this unit gets started, other units listed are also
    started; if any of other unit fails to start, this unit is not started
  - `Conflicts=`: if this unit gets started, other units listed are stopped
  - `Before=`: if this unit gets started at the same time with other units
    listed, this unit is started before other units
    - this does not express dependency; use `Wants=` for that
    - on stop, other units are stopped before this unit
  - `After=`: the opposite of `Before=`

## Specifiers

- `%g` and `%G` are group name and gid of the systemd instance
- `%h` is the home dir of the systemd instance
- `%H` is the hostname
- `%i` is the instance name, the string between the first `@` and the type
  suffix
- `%n` and `%N` are the unit name, with or without the type suffix
- `%p` is the string before the first `@`, or `%N`
- `%s` is the shell of the systemd instance
- `%t` is the runtime dir (`/run` or `/run/users/$UID`) of the systemd
  instance
- `%u` and `%U` are user name and uid of the systemd instance

## Scopes and Slices

- `man systemd.scope`
  - scopes manage externally-created processes
    - contrary to processes created by service units 
  - `systemctl status init.scope` has pid1, which is externally-created
  - `systemctl --user status init.scope` has the systemd user instance, which
    is externally-created
  - `systemctl status session-c1.scope` has `login` and processes forked from
    it, which are externally-created
- `man systemd.slice`
  - slices manage resources for processes (that is, scopes and services)
    hierarchically
  - `systemctl status -- -.slice` is the root slice and has
    - `init.scope` for pid1
    - `machine.slice` for vms/containers
    - `system.slice` for service units
    - `user.slice` for per-user slices
  - `systemctl status machine.slice` is the machine slice
    - a podman container creates two scopes in the machine slice, one for
      `conmon` that manages the container and one for the processes in the
      container
  - `systemctl status system.slice` is the system slice and consists of
    service units
  - `systemctl status user.slice` consists of per-user slices
    (`user-$UID.slice`)
    - `systemctl status user-$UID.slice` has
      - `session-c1.scope`
      - `user@$UID.service`
  - the systemd user instance is similar but different
    - `systemctl --user status -- -.slice` is actually
      `/user.slice/user-$UID.slice/user@$UID.service` and has
      - `init.scope` for the systemd user instance
      - `app.slice` is the defaut slice for service units
      - `session.slice` should be used for essential service units such as
        dbus

## Service Units

- `man systemd.service`
- `Type=` specifies when the manager considers the service ready
  - to start a service, the manager process calls `fork` to create a process
    which calls `execve` to execute the service binary
    - the forked process is considered the main process except for `forking`
  - `simple` means after the manager process calls `fork`
    - use `exec` instead
  - `exec` means after the forked process calls `execve`
  - `forking` means after the forked process terminates
    - the forked process is expected to fork again (i.e., call `daemon`)
    - the main process should be specified by `PIDFile=`, otherwise it is
      whatever process forked by the forked process
  - `oneshot` means after the forked process terminates
    - the main process is the forked process and has terminated
    - `RemainAfterExit=yes` should usually be set
  - `dbus` means after the dbus name is acquired
    - `BusName=` must be specified
  - `notify` means after `sd_notify("READY=1")` is received
  - `notify-reload` is similar to `notify`, except the service also supports
    reloading using signals
- `ExecStart=` is the command(s) to execute when starting the service
  - `ExecCondition=` and `ExecStartPre=` are optional and are executed before
    `ExecStart=`
  - `ExecStartPost=` is optional and is executed after `ExecStart=`
- `ExecStop=` is the command(s) to execute when stopping the service
  - this is optional and all remaining processes are always killed
  - `ExecStopPost=` is optional and is executed after `ExecStop=`
    - it can be executed without `ExecStop=` when service startup failed

## Timer Units

- there is `at`
  - <http://blog.calhariz.com/index.php/tag/at>
    - git repo <https://salsa.debian.org/debian/at>
  - a user invokes `at` to submit a job
    - `at` is suid and is owned by `daemon`
      - that is, euid will be `daemon` when executing
      - but `at` calls `setreuid` immediately to swap euid and ruid
    - it checks `/etc/at.allow` and `/etc/at.deny` to determine if user can
      submit jobs
      - `man 2 access` says fs perm is based on ruid
    - it creates a file (job) under `/var/spool/cron/atjobs`
      - `man 2 open` says the euid will be the file owner
      - the contents of the file is a script to
        - a comment that contains uid/gid/mail
        - restore the env
        - cd to cwd
        - execute the user-speicified commands
    - it sends `SIGHUP` to wake up atd
      - `/run/atd.pid` has the pid
  - `atd` is a daemon
    - it runs as `root` but set euid to `daemon`
    - it sleeps inside a main loop most of the time
    - when woken up, it scans `/var/spool/cron/atjobs`
      - for each job that is ready, it forks a child to run the job
      - it then sleeps until the next job is ready
    - when running a job in a child,
      - it parses the comment at the beginning for uid/gid/mail
      - it `setuid`/`setgid` to become the uid/gid
      - it invokes `/bin/sh` to run the script and redirects stdout/stderr to
        a file under `/var/spool/cron/atspool`
      - it invokes `sendmail -i <user>` to send the stdout/stderr to the user
- there is `cron`
  - `crontab` updates `/var/spool/cron/crontabs/<user>`
    - it is setgid
    - it checks `/etc/cron.allow` and `/etc/cron.deny` to determine if user
      can use cron
  - `cron` is a daemon
    - it scans `/var/spool/cron/crontabs` for user crontabs
    - it also scans `/etc/cron.d` and `/etc/crontabs` for system crontabs
    - there are usually system crontabs to run all executables under these
      directories periodically
      - `/etc/cron.daily`
      - `/etc/cron.hourly`
      - `/etc/cron.weekly`
      - `/etc/cron.monthly`
- systemd provides timers
  - `foo.timer`
    - `[Timer]` section defines when does the timer activate and which unit to
      activate
