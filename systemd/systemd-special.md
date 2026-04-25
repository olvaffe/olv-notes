# systemd special

## Bootup

- `man bootup`
- `systemctl start default.target`
  - this is the default on boot, unless overriden by `systemd.unit=`
  - `systemctl set-default <unit>.target` sets the default unit
    - it is `graphical.target` by default
    - the other choice is `multi-user.target`
- `systemctl start <poweroff|reboot>.target`
  - this powers off or reboots the machine
  - fwiw, `halt.target` halts the system, making it safe to cut the power but
    does not cut the power
- `systemctl list-dependencies graphical.target`, focusing on target units
  - `graphical.target`, `Requires=multi-user.target`
    - `multi-user.target`, `Requires=basic.target`
      - `basic.target`, `Requires=sysinit.target`, `Wants=sockets.target timers.target paths.target slices.target`
        - `paths.target`
        - `slices.target`
        - `sockets.target`
        - `sysinit.target`, `Wants=local-fs.target swap.target`
          - `cryptsetup.target`, meson symlink in `sysinit.target.wants/`
          - `integritysetup.target`, meson symlink in `sysinit.target.wants/`
          - `local-fs.target`
          - `swap.target`
          - `veritysetup.target`, meson symlink in `sysinit.target.wants/`
        - `timers.target`
      - `getty.target`, meson symlink in `multi-user.target.wants/`
      - `machines.target`, `WantedBy=multi-user.target`
      - `remote-fs.target`, `WantedBy=multi-user.target`
- `systemctl list-dependencies graphical.target`, focusing on service units
  - `graphical.target`
    - `display-manager.service`
      - e.g., `gdm.service` has `Alias=display-manager.service` to create the
        symlink
      - some distros remove the line and manage the symlink in their package
        managers instead
    - `multi-user.target`
      - most services
      - `basic.target`
        - `sysinit.target`
          - boot-related services
- `systemctl list-dependencies poweroff.target`
  - `poweroff.target`, `Requires=systemd-poweroff.service`
    - `systemd-poweroff.service`, `Requires=shutdown.target umount.target final.target`
      - `shutdown.target`
      - `umount.target`
      - `final.target`
  - how this works is
    - all service units by default has `Conflicts=shutdown.target`, causing
      them to be stopped
    - all mount units by default has `Conflicts=umount.target`, causing them
      to be umounted
    - `final.target` does not appear to be used
- initrd
  - when there is `/etc/initrd-release` (e.g., in initramfs), the default
    target is `initrd.target`
  - `sysinit.target` is reached after journal, udev, vconsole, etc.
  - `basic.target`
  - `initrd-root-device.target` is reached after root device appears
    - `systemd-fstab-generator` parses `root=` to generate root `.device`
  - `initrd-root-fs.target` is reached after root mounted
    - `systemd-fstab-generator` parses `root=` to generate `sysroot.mount`
  - `initrd-fs.target` is reached after all fs mounted
    - `initrd-parse-etc.service` parses `/sysroot/etc/fstab` to generate
      `.mount`
    - it also starts `initrd-cleanup.service` to isolate to
      `initrd-switch-root.target`
  - `initrd.target`
  - `initrd-switch-root.target`
    - it starts `initrd-switch-root.service` to `systemctl switch-root`
  - many units have `OnFailure=emergency.target` to drop to emergency sulogin

## Built-in Units

- systemd comes with a lot of built-in units
  - they are under `units/` in the source tree
- `presets/90-systemd.preset` specifies preset
  - on first boot, `manager_preset_all` applies `UNIT_FILE_PRESET_ENABLE_ONLY`
    automatically
  - these targets are enabled
    - `machines.target`, `WantedBy=multi-user.target`
    - `reboot.target`, `Alias=ctrl-alt-del.target`
    - `remote-cryptsetup.target`, `WantedBy=multi-user.target`
    - `remote-fs.target`, `WantedBy=multi-user.target`
- `units/meson.build` also creates symlinks for the built-in units
  - `cryptsetup.target` under `sysinit.target.wants/`
  - `dev-hugepages.mount` under `sysinit.target.wants/`
  - `dev-mqueue.mount` under `sysinit.target.wants/`
  - `getty.target` under `multi-user.target.wants/`
  - `getty@.service.in` as `autovt@.service`
  - `graphical.target` as `default.target`
  - `systemd-battery-check.service.in` under `initrd.target.wants/`
  - `systemd-bsod.service.in` under `initrd.target.wants/`
  - `integritysetup.target` under `sysinit.target.wants/`
  - `kmod-static-nodes.service.in` under `sysinit.target.wants/`
  - `ldconfig.service` under `sysinit.target.wants/`
  - `proc-sys-fs-binfmt_misc.automount` under `sysinit.target.wants/`
  - `reboot.target` as `ctrl-alt-del.target`
  - `remote-cryptsetup.target` under `initrd-root-device.target.wants/`
  - `remote-integritysetup.target` under `initrd-root-device.target.wants/`
  - `remote-veritysetup.target` under `initrd-root-device.target.wants/`
  - `sys-fs-fuse-connections.mount` under `sysinit.target.wants/`
  - `sys-kernel-config.mount` under `sysinit.target.wants/`
  - `sys-kernel-debug.mount` under `sysinit.target.wants/`
  - `sys-kernel-tracing.mount` under `sysinit.target.wants/`
  - `systemd-ask-password.socket` under `sockets.target.wants/`
  - `systemd-ask-password-console.path` under `sysinit.target.wants/`
  - `systemd-ask-password-wall.path` under `multi-user.target.wants/`
  - `systemd-binfmt.service.in` under `sysinit.target.wants/`
  - `systemd-boot-random-seed.service` under `sysinit.target.wants/`
  - `systemd-bootctl.socket` under `sockets.target.wants/`
  - `systemd-confext-initrd.service` under `initrd.target.wants/`
  - `systemd-coredump.socket` under `sockets.target.wants/`
  - `systemd-creds.socket` under `sockets.target.wants/`
  - `systemd-factory-reset.socket` under `sockets.target.wants/`
  - `systemd-factory-reset-request.service.in` under `factory-reset.target.wants/`
  - `systemd-firstboot.service` under `sysinit.target.wants/`
  - `systemd-hibernate-clear.service.in` under `sysinit.target.wants/`
  - `systemd-hostnamed.service.in` as `dbus-org.freedesktop.hostname1.service`
  - `systemd-hostnamed.socket` under `sockets.target.wants/`
  - `systemd-hwdb-update.service.in` under `sysinit.target.wants/`
  - `systemd-importd.service.in` as `dbus-org.freedesktop.import1.service`
  - `systemd-importd.socket` under `sockets.target.wants/`
  - `imports.target` under `sysinit.target.wants/`
  - `systemd-initctl.socket` under `sockets.target.wants/`
  - `systemd-journal-catalog-update.service` under `sysinit.target.wants/`
  - `systemd-journal-flush.service` under `sysinit.target.wants/`
  - `systemd-journald-dev-log.socket` under `sockets.target.wants/`
  - `systemd-journald.service.in` under `sysinit.target.wants/`
  - `systemd-journald.socket` under `sockets.target.wants/`
  - `systemd-localed.service.in` as `dbus-org.freedesktop.locale1.service`
  - `systemd-logind.service.in` under `multi-user.target.wants/`
  - `systemd-logind.service.in` as `dbus-org.freedesktop.login1.service`
  - `systemd-machine-id-commit.service` under `sysinit.target.wants/`
  - `systemd-machined.service.in` as `dbus-org.freedesktop.machine1.service`
  - `systemd-machined.socket` under `sockets.target.wants/`
  - `systemd-modules-load.service.in` under `sysinit.target.wants/`
  - `systemd-pcrextend.socket` under `sockets.target.wants/`
  - `systemd-pcrmachine.service.in` under `sysinit.target.wants/`
  - `systemd-pcrphase-factory-reset.service.in` under `factory-reset.target.wants/`
  - `systemd-pcrphase-initrd.service.in` under `initrd.target.wants/`
  - `systemd-pcrphase-sysinit.service.in` under `sysinit.target.wants/`
  - `systemd-pcrphase-storage-target-mode.service.in` under `storage-target-mode.target.wants/`
  - `systemd-pcrphase.service.in` under `sysinit.target.wants/`
  - `systemd-tpm2-setup.service.in` under `sysinit.target.wants/`
  - `systemd-tpm2-setup-early.service.in` under `sysinit.target.wants/`
  - `systemd-pcrlock.socket` under `sockets.target.wants/`
  - `systemd-portabled.service.in` as `dbus-org.freedesktop.portable1.service`
  - `systemd-random-seed.service.in` under `sysinit.target.wants/`
  - `systemd-repart.service` under `sysinit.target.wants/`
  - `systemd-repart.service` under `initrd-root-fs.target.wants/`
  - `systemd-sysctl.service.in` under `sysinit.target.wants/`
  - `systemd-sysext-initrd.service` under `initrd.target.wants/`
  - `systemd-sysext.socket` under `sockets.target.wants/`
  - `systemd-sysupdated.service.in` as `dbus-org.freedesktop.sysupdate1.service`
  - `systemd-sysusers.service` under `sysinit.target.wants/`
  - `systemd-timedated.service.in` as `dbus-org.freedesktop.timedate1.service`
  - `systemd-tmpfiles-clean.timer` under `timers.target.wants/`
  - `systemd-tmpfiles-setup-dev-early.service` under `sysinit.target.wants/`
  - `systemd-tmpfiles-setup-dev.service` under `sysinit.target.wants/`
  - `systemd-tmpfiles-setup.service` under `sysinit.target.wants/`
  - `systemd-udev-trigger.service` under `sysinit.target.wants/`
  - `systemd-udevd-control.socket` under `sockets.target.wants/`
  - `systemd-udevd-kernel.socket` under `sockets.target.wants/`
  - `systemd-udevd-varlink.socket` under `sockets.target.wants/`
  - `systemd-udevd.service.in` under `sysinit.target.wants/`
  - `systemd-update-done.service.in` under `sysinit.target.wants/`
  - `systemd-update-utmp-runlevel.service.in` under `multi-user.target.wants/`
  - `systemd-update-utmp-runlevel.service.in` under `graphical.target.wants/`
  - `systemd-update-utmp-runlevel.service.in` under `rescue.target.wants/`
  - `systemd-update-utmp.service.in` under `sysinit.target.wants/`
  - `systemd-user-sessions.service.in` under `multi-user.target.wants/`
  - `tmp.mount` under `local-fs.target.wants/`
  - `var-lib-machines.mount` under `remote-fs.target.wants/`
  - `var-lib-machines.mount` under `machines.target.wants/`
  - `veritysetup.target` under `sysinit.target.wants/`

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
  - `systemd-makefs`, if enabled in `/etc/fstab`, creates a fs on demand
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

## Special Units

- `man systemd.special`
- active and passive target units
  - `network-online.target` is an active target unit
    - `systemctl list-dependencies network-online.target` lists providers
      - its providers have `WantedBy=network-online.target` and
        `Before=network-online.target`
    - `systemctl list-dependencies --reverse network-online.target` lists
      consumers
      - its consumers have `Wants=network-online.target` and
        `After=network-online.target`
    - the idea is, when there are N providers and M consumers,
      - `systemctl start network-online.target` pulls in providers
      - consumers also pull in providers automatically
  - `network.target` is a passive target unit
    - `systemctl list-dependencies --reverse network.target` list providers
      - its producers have `Wants=network.target` and `Before=network.target`
    - `systemctl list-dependencies network.target` is empty
      - its consumers only have `After=network.target`
    - the idea is, when there are N providers and M consumers,
      - `systemctl start network.target` pulls in no producer
        - in fact, passive target units have `RefuseManualStart=yes`
      - consumers do not pull in providers either
      - the passive target merely defines an order for the providers and
        consumers
      - providers and consumers are pulled in via other means
- targets
  - `basic.target`
  - `bluetooth.target`
  - `cryptsetup-pre.target`
  - `veritysetup.target`
  - `ctrl-alt-del.target`
  - `boot-complete.target`
  - `default.target`
  - `emergency.target`
    - only `init` and `sulogin`
    - `local-fs.target` has `OnFailure=emergency.target` to enter emergency
      target on any mount failure
  - `exit.target`
  - `factory-reset.target`
  - `factory-reset-now.target`
  - `final.target`
  - `getty.target`
  - `graphical.target`
  - `halt.target`
  - `hibernate.target`
  - `hybrid-sleep.target`
  - `suspend-then-hibernate.target`
  - `initrd.target`
  - `initrd-fs.target`
  - `initrd-root-device.target`
  - `initrd-root-fs.target`
  - `initrd-usr-fs.target`
  - `integritysetup-pre.target`
  - `integritysetup.target`
  - `kbrequest.target`
  - `kexec.target`
  - `local-fs.target`
  - `machines.target`
  - `multi-user.target`
  - `network-online.target`
  - `paths.target`
  - `poweroff.target`
  - `printer.target`
  - `reboot.target`
  - `remote-cryptsetup.target`
  - `remote-integritysetup.target`
  - `remote-veritysetup.target`
  - `remote-fs.target`
  - `rescue.target`
    - this is single-user target
    - comparing to `multi-user.target`,
      - it requires `sysinit.target` but not `basic.target`
      - most services are `WantedBy=multi-user.target` and are not started
  - `runlevel2.target`
  - `runlevel3.target`
  - `runlevel4.target`
  - `runlevel5.target`
  - `shutdown.target`
  - `sigpwr.target`
  - `sleep.target`
  - `slices.target`
  - `smartcard.target`
  - `sockets.target`
  - `soft-reboot.target`
  - `sound.target`
  - `storage-target-mode.target`
  - `suspend.target`
  - `swap.target`
  - `sysinit.target`
  - `system-update.target`
  - `system-update-pre.target`
  - `timers.target`
  - `tpm2.target`
  - `umount.target`
  - `usb-gadget.target`
- special passive target units
  - `blockdev@.target`
  - `cryptsetup.target`
  - `veritysetup-pre.target`
  - `first-boot-complete.target`
    - e.g., `systemd-firstboot.service` pulls it in
    - consumers can have `After=first-boot-complete.target`
  - `getty-pre.target`
  - `local-fs-pre.target`
  - `network.target`
    - e.g., `systemd-networkd.service` pulls it in
    - consumers can have `After=network.target`
  - `network-pre.target`
  - `nss-lookup.target`
    - e.g., `systemd-resolved.service` pulls it in
    - consumers can have `After=nss-lookup.target`
  - `nss-user-lookup.target`
  - `remote-fs-pre.target`
  - `rpcbind.target`
  - `ssh-access.target`
  - `time-set.target`
    - e.g., `systemd-timesyncd.service` pulls it in
    - consumers can have `After=time-set.target`
  - `time-sync.target`
    - e.g., `systemd-time-wait-sync.service` pulls it in
    - consumers can have `After=time-sync.target`
- special slice units
  - `-.slice`
  - `machine.slice`
  - `capsule.slice`
  - `system.slice`
  - `user.slice`
- rest
  - `-.mount`
  - `dbus.service`
  - `dbus.socket`
  - `display-manager.service`
    - e.g., `gdm.service` has `Alias=display-manager.service`
  - `init.scope`
  - `syslog.socket`
  - `system-update-cleanup.service`
