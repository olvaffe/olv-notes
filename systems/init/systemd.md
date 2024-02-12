systemd
=======

## Booting

- `man bootup`
  - systemd will activate `default.target` on boot
    - it is usually a symlink to `graphical.target` or `multi-user.target`
  - systemd will activate `poweroff.target` or `reboot.target` on shutdown
    - fwiw, `halt.target` halts the system, making it safe to cut the power
      but does not cut the power
- `systemctl list-dependencies graphical.target`
  - it depends on `multi-user.target` and a few other services
  - `display-manager.service` is one of the services, which is usually a
    symlink to `gdm.service`
- `systemctl list-dependencies multi-user.target`
  - it depends on several targets and many services
  - targets are `basic.target`, `getty.target`, `machines.target`, and
    `remote-fs.target`
  - services include `dbus.service`, `ssh.service`, `systemd-logind.service`,
    `systemd-user-sessions.service`, etc.
- `systemctl list-dependencies basic.target`
  - it depends on several targets and others
  - targets are `paths.target`, `slices.target`, `sockets.target`,
    `sysinit.target`, and `timers.target`
- `systemctl list-dependencies sysinit.target`
  - it depends on several targets and many services
  - targets are `cryptsetup.target`, `integritysetup.target`,
    `local-fs.target`, `swap.target`, and `veritysetup.target`
  - services include `systemd-journald.service`, `systemd-timesyncd.service`,
    `systemd-udevd.service`, etc.
- shutdown is a bit different
  - `poweroff.target` depends on `systemd-poweroff.service`
  - `systemd-poweroff.service` depends on `final.target`, `shutdown.target`,
    and `umount.target`
  - by default, a service unit has `DefaultDependencies=yes` which implies
    `Requires=sysinit.target` and `Conflicts=shutdown.target`
    - this means a service is stopped by default before `shutdown.target`

## Old SysVinit (`/sbin/init`)

- PID 1
- it parses `/etc/inittab`
  - usually, it runs scripts in `/etc/rcS.d` and `/etc/rc2.d`.  Then runs
    `getty` for `tty[1-6]`
- `rcS.d`
  - `S01mountkernfs.sh` mounts `/run`, `/proc`, `/sys`, etc.
  - `S02udev` makes sure `/dev` is mounted as devtmpfs and starts udev
  - `S03mountdevsubfs.sh` mounts `/run/shm`, `/dev/pts`, etc.
  - `S05keyboard-setup` runs `setupcon` to set up kernel keymap
    - It is a script that runs `loadkeys`
    - This is an early setup for, say, checkroot failure interaction
  - `S07checkroot.sh` remounts `/` according to `/etc/fstab`
  - `S08kmod` load modules listed in `/etc/modules`
  - `S10mountall.sh` mounts all fs listed in `/etc/fstab`
  - `S13procps` runs `sysctl` for settings listed in `/etc/sysctl.conf`
  - `S15networking` runs `ifup`, which reads `/etc/network/interfaces`
  - `S19console-setup` run `setupcon`
  - more
- `rc2.d`
  - `S01binfmt-support` updates kernel binfmt database
  - `S01rsyslog` starts syslogd
  - `S02dbus` starts system-wise dbus
  - `S02ssh` starts sshd
  - `S05gdm3` starts gdm3
  - more

## Binaries

- libexec binaries
  - `systemd` is pid 1
  - `systemd-backlight` saves/restores backlight brightness on boot/shutdown
    - values are stored in `/var/lib/systemd/backlight`
  - `systemd-battery-check` checks the battery level in early boot
    - it shows an error and powers off if the battery level is below 5% and is
      not charging
  - `systemd-binfmt` registers misc binfmt
    - it registers new binfmt through `/proc/sys/fs/binfmt_misc/register`
    - it reads the binfmt to be registered from
      - `/etc/binfmt.d`
      - `/run/binfmt.d`
      - `/usr/local/lib/binfmt.d`
      - `/usr/lib/binfmt.d`
      - `/lib/binfmt.d`
  - `systemd-bless-boot`, if enabled, marks the current boot successful (for
    A/B boot)
  - `systemd-boot-check-no-failures`, if enabled, stops before
    `boot-complete.target` if any service fails
  - `systemd-bsod` shows bsod if boot fails
  - `systemd-cgroups-agent` is used if cgroup v1
  - `systemd-coredump`, if enabled, is the kernel coredump handler
    - it is set up in `/proc/sys/kernel/core_pattern`
    - it saves coredumps to `/var/lib/systemd/coredump`
  - `systemd-cryptsetup`, if enabled, sets up disk encryption
  - `systemd-executor` is executed first to set up jails before exec'ing the
    real binary
  - `systemd-export` is invoked by `importd` to export an vm or container
    image
  - `systemd-fsck` is a wrapper for `fsck.<type>`
  - `systemd-growfs`, if enabled in `/etc/fstab`, resizes a fs on demand
  - `systemd-hibernate-resume` writes to `/sys/power/resume` to prep for
    hibernation
  - `systemd-homed`, if enabled, manages home dirs
  - `systemd-homework` is invoked by `homed` to manage home dirs
  - `systemd-hostnamed` manages system host names
    - it provides `org.freedesktop.hostname1`
    - it provides/stores additional info in `/etc/machine-info`
  - `systemd-import` is invoked by `importd` to import a vm/container image
  - `systemd-import-fs` is invoked by `importd` to import a vm/container dir
  - `systemd-importd` manages vm or container images
  - `systemd-integritysetup` sets up dm-integrity, to store checksums for each
    block
  - `systemd-journal-gatewayd` allows journals to be pulled by a remote server
  - `systemd-journal-remote` pulls or receives journals from remote clients
  - `systemd-journal-upload` pushes local journals to a remote server
  - `systemd-journald` manages system journals
  - `systemd-localed` manages system locale
    - it provides `org.freedesktop.locale1`
  - `systemd-logind` manage system logins
    - it provides `org.freedesktop.login1`
  - `systemd-machined` manages vms and containers
  - `systemd-makefs`, inf enabled in `/etc/fstab`, creates a fs on demand
  - `systemd-measure` is invoked by `ukify` for kernel signing
  - `systemd-modules-load` loads modules during boot
    - it is based on `libkmod`.
    - It scans these directories to find modules to load
      - `/etc/modules-load.d`
      - `/run/modules-load.d`
      - `/usr/local/lib/modules-load.d`
      - `/usr/lib/modules-load.d`
      - `/lib/modules-load.d`
  - `systemd-network-generator` generates `networkd` configs from kernel
    cmdline
  - `systemd-networkd` manages system network
  - `systemd-networkd-wait-online` blocks until the system is online
  - `systemd-oomd` is a userspace oom killer based on cgroup and psi
  - `systemd-pcrextend` is related to tpm
  - `systemd-pcrlock` is related to tpm
  - `systemd-portabled` manages portable services (chroots that provide
    services?)
  - `systemd-pstore` moves dumps in `/sys/fs/pstore` to
    `/var/lib/systemd/pstore` on boot
  - `systemd-pull` is invoked by `importd` to download a vm/container image
  - `systemd-quotacheck` is a wrapper to `quotacheck`
  - `systemd-random-seed` saves and restores `/dev/urandom` on boot/shutdown
  - `systemd-remount-fs` remounts fs according to `/etc/fstab`
  - `systemd-reply-password` is invoked by a password agent to write password
    over a socket
  - `systemd-resolved` manages system name resolving
  - `systemd-rfkill` saves/restores rfkill state in `/var/lib/systemd/rfkill`
  - `systemd-shutdown` is execv'd by pid 1 on shutdown
  - `systemd-sleep` is invoked during hibernate/suspend
    - it run executables in `/lib/systemd-sleep`, writes to `/sys/power/state`
      to suspend or hibernate, and runs executables in `/lib/systemd-sleep`
      again with different argument
  - `systemd-socket-proxyd` is a socket proxy for socket activation
  - `systemd-storagetm` exports all block devices as NVMe-TCP devices, to be
    attached to a remote host
  - `systemd-sulogin-shell` is invoked during recovery
  - `systemd-sysctl` applies sysctl on boot
    - it reads these files to set sysctl
      - `/etc/sysctl.d`
      - `/run/sysctl.d`
      - `/usr/local/lib/sysctl.d`
      - `/usr/lib/sysctl.d`
      - `/lib/sysctl.d`
      - `/etc/sysctl.conf`
  - `systemd-sysroot-fstab-check` parses `<sysroot>/etc/fstab` and does a
    `daemon-reload` from initrd
  - `systemd-sysupdate` applies systemd updates according to
    `/etc/sysupdate.d`
  - `systemd-time-wait-sync` blocks until the system time is synced
  - `systemd-timedated` manages system time
    - it provides `org.freedesktop.timedate1` to set time, timezone, ntp, and
      etc.
  - `systemd-timesyncd` is an ntp client
  - `systemd-tpm2-setup` is for tpm
  - `systemd-udevd` is udev daemon
  - `systemd-update-done` updates `/etc/.updated` and `/var/.updated`
  - `systemd-update-utmp` records boot/shutdown to utmp
  - `systemd-user-runtime-dir` creates/removes `/run/user/$UID`
  - `systemd-user-sessions` creates/removes `/run/nologin` to disable user
    login during boot/shutdown
  - `systemd-userdbd`, if enabled, provides the user db
  - `systemd-userwork` is invoked by `userdbd`
  - `systemd-vconsole-setup` is a wrapper for `loadkeys` and `setfont`
  - `systemd-veritysetup` sets up dm-verity
  - `systemd-volatile-root` populates a tmpfs and mounts it to `/` (and mounts
    `/usr` read-only), creating a stateless system
  - `systemd-xdg-autostart-condition` is invoked by `xdg-autostart-generator`
    to control autostart based on `XDG_CURRENT_DESKTOP`
- bin binaries
  - `bootctl` controls uefi and `systemd-boot`
  - `busctl` talks to dbus
  - `coredumpctl` retrieves saved coredumps and their metadata
  - `homectl` talks to `homed`
  - `hostnamectl` talks to `hostnamed`
  - `journalctl` talks to `journald`
  - `kernel-install` installs the specified kernel
    - kernel `make install` invokes `installkernel`, which can be a symlink to
      `kernel-install`
  - `localectl` talks to `localed`
  - `loginctl` talks to `logind`
  - `machinectl` talks to `machined`
  - `mount.ddi` is a symlink to `systemd-dissect`
  - `networkctl` talks to `networkd`
  - `oomctl` talks to `oomd`
  - `portablectl` talks to `portabled`
  - `resolvectl` talks to `resolved`
  - `systemctl` talks to `systemd`
  - `systemd-ac-power` checks if the system is on ac
  - `systemd-analyze` analyzes systemd stats
  - `systemd-ask-password` queries system passwords interactively and prints
    them to stdout
  - `systemd-cat` logs its stdin to `journald`
  - `systemd-cgls` lists cgroups
  - `systemd-cgtop` monitors cgroups
  - `systemd-confext` activates system extension images for `/etc`
  - `systemd-creds` encrypts/descrypts service credentials
  - `systemd-cryptenroll` enrolls hw security tokens
  - `systemd-cryptsetup` is a wrapper to `cryptsetup`
  - `systemd-delta` shows overriden config files
  - `systemd-detect-virt` checks if the system is in a vm/container
  - `systemd-dissect` dissects discoverable system images (ddi)
  - `systemd-escape` escapes strings
  - `systemd-firstboot` initializes basic configs on first boot
  - `systemd-hwdb` queried udev hwdb
  - `systemd-id128` generates/queries sd-128 ids
  - `systemd-inhibit` inhibits shutdown/sleep/etc.
  - `systemd-machine-id-setup` generates `/etc/machine-id`
  - `systemd-mount` schedules a mount through `systemd`
  - `systemd-notify` is a wrapper for `sd_notify`
  - `systemd-nspawn` spawns a container
  - `systemd-path` lists system paths
  - `systemd-repart` repartitions according to `/etc/repart.d`
  - `systemd-resolve` is a symlink to `resolvectl`
  - `systemd-run` schedules a cmd through `systemd`
  - `systemd-socket-activate` tests socket activation
  - `systemd-stdio-bridge` is a proxy between its stdin/stdout and dbus
    - when executed over ssh, it allows remote dbus calls
  - `systemd-sysext` activates system extension images
  - `systemd-sysusers` creates system users based on `/etc/sysusers.d`
  - `systemd-tmpfiles` creates tmp files/dirs based on `/etc/tmpfiles.d`
  - `systemd-tty-sk-password-agent` is a tty-based password agent
    - <https://systemd.io/PASSWORD_AGENTS/>
  - `systemd-umount` is a symlink to `systemd-mount` to umount
  - `systemd-vmspawn` spawns a vm
  - `timedatectl` talks to `timedated`
  - `udevadm` talks to `udevd`
  - `userdbctl` talks to `userdbd`
  - `varlinkctl` introspects varlink services
- others
  - `systemd-fstab-generator`
    - it scans `/etc/fstab` to convert it into swap, mount, and/or automount
      units
    - the unit files are written to `/tmp` by default
  - `systemd-getty-generator`
    - generates `serial-getty@.service` by looking at
      `/sys/class/tty/console/active` and containers
  - `systemd-rc-local-generator`
    - adds `rc-local.service` to `multi-user.target.wants` and
      `halt-local.service` to `final.target.wants`
  - `systemd-readahead-collect` and `systemd-readahead-replay`
    - the former collects files used during boot, and the latter caches the
      files
  - `systemd-system-update-generator`
    - <http://freedesktop.org/wiki/Software/systemd/SystemUpdates>
    - it checks `/system-update` and redirect systemd to a special system update
      target unit if exists.  This allows us to do Windows-like system update.
  - `systemd-initctl`
    - listens to fds listed in env var `LISTEN_FDS` to provide `/dev/initctl`
      emulation
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
