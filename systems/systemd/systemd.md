systemd
=======

## Booting

- systemd will activate `default.target` on boot
  - it is a symlink to `graphical.target`
  - it has this dependency tree

    graphical.target
     -> multi-user.target
       -> basic.target
         -> sysinit.target
           -> local-fs.target
           -> swap.target
         -> sockets.target

## Source tree

- `bash-completion/systemd-bash-completion.sh`
  - bash completion for systemctl and loginctl
- `keymaps/`
  - vendor keymaps used by udev `95-keymap.rules`
- `sysctl.d/coredump.conf`
  - make kernel execute `systemd-coredump` for core dumping
  - it gives systemd a chance to hook up core dumping with journal
- `tmpfiles.d/`
  - config files for `systemd-tmpfiles`
  - they define temporary files/directories to be created, removed, or cleaned
- `units` see units below

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
  - `Requires=`: if this unit gets started, other units listed are also
    started; if any of other unit fails to start, this unit is not started
  - `Conflicts=`: if this unit gets started, other units listed are stopped
  - `Before=`: if this unit gets started at the same time with other units
    listed, this unit is started before other units
    - this does not express dependency; use `Wants=` for that
    - on stop, other units are stopped before this unit
  - `After=`: the opposite of `Before=`

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

## Helpers

- `systemd-ac-power`
  - used libudev to detect if the system is on AC power
- `systemd-ask-password`
  - asks the user interactively for a password, and prints it to stdout
- `systemd-binfmt`
  - it registers new binfmt through `/proc/sys/fs/binfmt_misc/register`
  - it reads the binfmt to be registered from
    - `/etc/binfmt.d`
    - `/run/binfmt.d`
    - `/usr/local/lib/binfmt.d`
    - `/usr/lib/binfmt.d`
    - `/lib/binfmt.d`
- `systemd-fsck` is a wrapper to fsck
- `systemd-fstab-generator`
  - it scans `/etc/fstab` to convert it into swap, mount, and/or automount
    units
  - the unit files are written to `/tmp` by default
- `systemd-getty-generator`
  - generates `serial-getty@.service` by looking at
    `/sys/class/tty/console/active` and containers
- `systemd-modules-load`
  - it is based on `libkmod`.
  - It scans these directories to find modules to load
    - `/etc/modules-load.d`
    - `/run/modules-load.d`
    - `/usr/local/lib/modules-load.d`
    - `/usr/lib/modules-load.d`
    - `/lib/modules-load.d`
- `systemd-quotacheck` is a wrapper to quotacheck
- `systemd-random-seed` reads or writes a random seed from or to
  `/dev/urandom`
- `systemd-rc-local-generator`
  - adds `rc-local.service` to `multi-user.target.wants` and
    `halt-local.service` to `final.target.wants`
- `systemd-readahead-collect` and `systemd-readahead-replay`
  - the former collects files used during boot, and the latter caches the
    files
- `systemd-remount-fs`
  - remounts root and kernel filesystems (such as proc, sys, and etc.)
- `systemd-reply-password`
  - reads password from stdin and sends it to a socket
- `systemd-shutdownd`
  - run by systemd for delayed shutdown
- `systemd-sleep`
  - run executables in `/lib/systemd-sleep`, write to `/sys/power/state` to
    suspend or hibernate, and run executables in `/lib/systemd-sleep` again
    with different argument
- `systemd-stdio-bridge`
  - allows one to talk to D-Bus system bus from stdio
  - we can open an SSH tunnel to execute it.  It allows remote connection to
    the system bus
- `systemd-sysctl`
  - it reads these files to set sysctl
    - `/etc/sysctl.d`
    - `/run/sysctl.d`
    - `/usr/local/lib/sysctl.d`
    - `/usr/lib/sysctl.d`
    - `/lib/sysctl.d`
    - `/etc/sysctl.conf`
- `systemd-system-update-generator`
  - <http://freedesktop.org/wiki/Software/systemd/SystemUpdates>
  - it checks `/system-update` and redirect systemd to a special system update
    target unit if exists.  This allows us to do Windows-like system update.
- `systemd-tmpfiles`
  - it searches its config files in this order
    - `/etc/tmpfiles.d`
    - `/run/tmpfiles.d`
    - `/usr/local/lib/tmpfiles.d`
    - `/usr/lib/tmpfiles.d`
  - it then creates, removes, or cleans temporary files as defined by
    the config files
- `systemd-update-utmp`
  - update utmp for boot/reboot/shutdown messages
- `systemd-vconsole-setup`
  - setup a virtual console (keymap, font, and etc.)

## Tools

- `systemd-analyze`
  - it analyzes systemd log to, for example, show how long it took to start a
    service
- `systemd-cgls`
  - shows cgroup contents recursively
- `systemd-delta` shows which config from which directory overrides another
  - the prefixes are, in this order,
    - `/etc`
    - `/run`
    - `/usr/local/lib`
    - `/usr/local/share`
    - `/usr/lib`
    - `/usr/share`
    - `/lib`
  - the suffixes are
    - `sysctl.d`
    - `tmpfiles.d`
    - `modules-load.d`
    - `binfmt.d`
    - `systemd/system`
    - `systemd/user`
    - `systemd/system.preset`
    - `systemd/user.preset`
    - `udev/rules.d`
    - `modprobe.d`
- `journalctl`
- `systemd-cat`
  - redirects its stdin to journal
- `loginctl`
- `systemd-machine-id-setup`
  - generates `/etc/machine-id`
- `systemd-notify`
  - shell script version for `sd_notify()`, which is used by daemons to notify
    systemd daemons' current descriptive states
- `systemd-nspawn`
  - spawns a namespace container
- `systemctl`
- `systemd-timestamp`
  - prints current (high-precision) time to stdout.  Can be used in a script
    for simple profiling
- `systemd-tty-ask-password-agent`
  - <http://www.freedesktop.org/wiki/Software/systemd/PasswordAgents>
- `bootctl` shows, lists, installs, removes EFI boot loaders
  - it can install `systemd-boot`, a simple EFI boot loader
- `busctl` lists and talks to D-Bus services
- `coredumpctl` lists saved coredumps
  - core dumps are piped to `/proc/sys/kernel/core_pattern`, which is set to
    `systemd-coredump`
  - man core(5)
- `homectl` to manage `systemd-homed`-managed users
- `hostnamectl` sets `/etc/hostname` and others
- `localectl` sets, gets, and lists locales
- `networkctl` talks to `systemd-networkd`
- `timedatectl`

## Daemons

- `systemd-hostnamed`
  - is a daemon providing `org.freedesktop.hostname1` to set and get hostname
  - `systemd-hostnamed` provides/stores additional info in `/etc/machine-info`
- `systemd-initctl`
  - listens to fds listed in env var `LISTEN_FDS` to provide `/dev/initctl`
    emulation
- `systemd-journald`
  - a syslog replacement
- `systemd-localed`
  - <http://www.freedesktop.org/wiki/Software/systemd/localed>
  - is a daemon providing `org.freedesktop.locale1` to set/get locale and
    console keyboard mappings
- `systemd-logind`
  - <http://www.freedesktop.org/wiki/Software/systemd/multiseat>
  - is a daemon providing `org.freedesktop.login1` to ??
- `systemd`
  - `src/core/main.c`
- `systemd-timedated`
  - provides `org.freedesktop.timedate1` to set time, timezone, ntp, and etc.
- `udev`

## Libraries

- `libsystemd-daemon`
  - header `sd-daemon.h`
  - an API for writing new-style daemons
- `libsystemd-id128`
  - an API to generate 128-bit (/ 32-char / 16-byte) UUID
- `libsystemd-journal`
- `libsystemd-login`
- `pam_systemd`
- `libudev`
- `libgudev`
- internal libraries
  - `shared` subdirectory provides these internal libraries
    - `libsystemd-shared`
    - `libsystemd-dbus`
    - `libsystemd-label`
    - `libsystemd-logs`
    - `libsystemd-capability`
    - `libsystemd-audit`
    - `libsystemd-acl`
  - `core` provides internal `libsystemd-core`
  - `libudev` provides internal `libudev-private`
  - `udev` provides internal `libudev-core`

## suspend

- systemd-logind opens evdev devices for
  - `KEY_POWER`/`KEY_POWER2`: power key
  - `KEY_SLEEP`: suspend key
  - `KEY_SUSPEND`: hibernate key
  - `SW_LID`: lid opened/closed
  - `SW_DOCK`: machine docked/undocked
- when lid closed, `button_lid_switch_handle_action` is called
  - enumerate DRM connectors in sysfs to see if any external display connected
  - enumerate `/sys/class/power_supply` to see if AC is online
  - depending on whether there are external display or whether AC is online,
    different actions can be taken
    - such as ignore, poweroff, reboot, halt, suspend, lock, etc.
  - by default, 
    - when external displays are connected, lid closed is ignored
    - otherwise, suspend

## namespaces

- systemd makes use of linux namespaces
- `lsns -t mnt`
  - `systemd-udevd` is configured `PrivateMounts=yes`
  - `systemd-logind`, `colord`, `ntpd`, and `upowerd` are configured
    `PrivateTmp=yes`
  - `bluetoothd`, `NetworkManager` are configured `ProtectSystem=yes`
  - `irqbalance` is configured `ReadOnlyPaths=/`
- `lsns -t net`
  - `rtkit-daemon` is configured `PrivateNetwork=yes`
- `lsns -t user`
  - `upowerd` is configured `PrivateUsers=yes`
- `lsns -t uts`
  - `systemd-udevd` and `systemd-logind` are configured `ProtectHostname=yes`
- ipc, pid, cgroup, time
  - no used by systemd on my system
  - but chrome makes use of net, user, pid
