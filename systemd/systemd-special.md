systemd special
===============

## Built-in Units

- systemd comes with a lot of built-in units
  - they are under `units/` in the source tree
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
    - `gdm.service` has `Alias=display-manager.service` to create the symlink
    - some distros remove the line and manage the symlink in their package
      managers instead
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

