Systemd Unit Configuration
==========================

## Units

- a unit file encodes information about a service, socket, device, mount,
  automount, swap, target, path, timer, etc.
- common options
  - `man systemd.unit` documents the `[Unit]` and `[Install]` sections common
    to all unit files
  - `man systemd.exec` documents exec options common to several sections
  - `man systemd.kill` documents killing options common to several sections
  - `man systemd.resource-control` documents resource control options common
    to several sections
  - `man systemd.directives` lists which options are documented in which man
    pages
- `man systemd.automount` documents the `[Automount]` section
- `man systemd.device` documents the `[Device]` section
  - there is no device-specific options
- `man systemd.mount` documents the `[Mount]` section
- `man systemd.path` documents the `[Path]` section
- `man systemd.scope` documents the `[Scope]` section
- `man systemd.service` documents the `[Service]` section
- `man systemd.socket` documents the `[Socket]` section
- `man systemd.slice` documents the `[Slice]` section
  - there is no slice-specific options
- `man systemd.swap` documents the `[Swap]` section
- `man systemd.target` documents the `[Target]` section
  - there is no target-specific options
- `man systemd.timer` documents the `[Timer]` section
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

## `[Unit]` Section

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


## `[Install]` Section

- `[Install]` is not used by `systemd`, but by `systemctl` for enable/disable
- `Alias=` lists additional symlinks to create/remove
- `WantedBy=`, `RequiredBy=`, and `UpheldBy=` list additoinal symlinks to be
  created/removed under `.wants/`, `.requires/`, and `.upholds/`
- `Also=` lists additional units to install/uninstall
- `DefaultInstance=` lists the default instance to be created/removed for a
  template unit file
- e.g., `systemctl enable systemd-networkd` installs
  - symlink
    `/etc/systemd/system/multi-user.target.wants/systemd-networkd.service` to
    `/usr/lib/systemd/system/systemd-networkd.service` because of `WantedBy=`
  - symlink `/etc/systemd/system/dbus-org.freedesktop.network1.service` to
    `/usr/lib/systemd/system/systemd-networkd.service` because of `Alias=`
  - symlink `/etc/systemd/system/sockets.target.wants/systemd-networkd.socket`
    to `/usr/lib/systemd/system/systemd-networkd.socket` because of `Also=`
  - symlink
    `/etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service`
    to `/usr/lib/systemd/system/systemd-networkd-wait-online.service` because
    of `Also=`
  - symlink
    `/etc/systemd/system/sysinit.target.wants/systemd-network-generator.service`
    to `/usr/lib/systemd/system/systemd-network-generator.service` because of
    `Also=`
- systemd package also creates symlinks for some units
  - they are created by `units/meson.build`
  - `man bootup`
  - `initrd-root-device.target.wants/`
    - `remote-cryptsetup.target`
    - `remote-veritysetup.target`
  - `initrd-root-fs.target.wants/`
    - `systemd-repart.service`
  - `initrd.target.wants/`
    - `systemd-battery-check.service`
    - `systemd-bsod.service`
    - `systemd-pcrphase-initrd.service`
  - `local-fs.target.wants/`
    - `tmp.mount`
  - `machines.target.wants/`
    - `var-lib-machines.mount`
  - `multi-user.target.wants/`
    - `getty.target`
    - `systemd-ask-password-wall.path`
    - `systemd-logind.service`
    - `systemd-user-sessions.service`
  - `remote-fs.target.wants/`
    - `var-lib-machines.mount`
  - `sockets.target.wants/`
    - `systemd-bootctl.socket`
    - `systemd-coredump.socket`
    - `systemd-creds.socket`
    - `systemd-hostnamed.socket`
    - `systemd-journald-dev-log.socket`
    - `systemd-journald.socket`
    - `systemd-pcrextend.socket`
    - `systemd-pcrlock.socket`
    - `systemd-sysext.socket`
    - `systemd-udevd-control.socket`
    - `systemd-udevd-kernel.socket`
  - `sysinit.target.wants/`
    - `cryptsetup.target`
    - `dev-hugepages.mount`
    - `dev-mqueue.mount`
    - `integritysetup.target`
    - `kmod-static-nodes.service`
    - `ldconfig.service`
    - `proc-sys-fs-binfmt_misc.automount`
    - `sys-fs-fuse-connections.mount`
    - `sys-kernel-config.mount`
    - `sys-kernel-debug.mount`
    - `sys-kernel-tracing.mount`
    - `systemd-ask-password-console.path`
    - `systemd-binfmt.service`
    - `systemd-boot-random-seed.service`
    - `systemd-firstboot.service`
    - `systemd-hibernate-clear.service`
    - `systemd-hwdb-update.service`
    - `systemd-journal-catalog-update.service`
    - `systemd-journal-flush.service`
    - `systemd-journald.service`
    - `systemd-machine-id-commit.service`
    - `systemd-modules-load.service`
    - `systemd-pcrmachine.service`
    - `systemd-pcrphase-sysinit.service`
    - `systemd-pcrphase.service`
    - `systemd-random-seed.service`
    - `systemd-repart.service`
    - `systemd-sysctl.service`
    - `systemd-sysusers.service`
    - `systemd-tmpfiles-setup-dev-early.service`
    - `systemd-tmpfiles-setup-dev.service`
    - `systemd-tmpfiles-setup.service`
    - `systemd-tpm2-setup-early.service`
    - `systemd-tpm2-setup.service`
    - `systemd-udev-trigger.service`
    - `systemd-udevd.service`
    - `systemd-update-done.service`
    - `systemd-update-utmp.service`
    - `veritysetup.target`
  - `timers.target.wants/`
    - `systemd-tmpfiles-clean.timer`

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
- `RestartSec=` specifies restart delay, default to 100ms
- `Restart=` specifies restart policy, default to no
  - `on-failure` restarts on failure

## Timer Units

- `man systemd.timer`
- `[Timer]` section defines when does the timer activate and which unit to
  activate
- there was `at`
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
- there was `cron`
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
