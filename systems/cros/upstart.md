Chrome OS upstart
=================

## Boot

- no initiramfs
- `root=PARTUUID=xxx/PARTNROFF=1 rootwait ro`
  - `PARTNROFF=1` chooses the next parition, which is useful for A/B rootfs
  - the rootfs is ext2
- `/sbin/init` is upstart and parses `/etc/init` for configurations
  - upstart makes sure essential device nodes (null, tty, console, kmsg, etc.)
    and mounts (devtmpfs, devpts, proc, sysfs) are created
  - it sends `startup` as the first event
- `pre-startup.conf` is triggered by the first event
  - it mounts /tmp, /run
  - it invokes `restorecon` for SELinux
- `udev.conf` is also triggered after `pre-startup.conf`
  - it triggers `udev-trigger-early.conf`
- `boot-splash.conf` is triggered after `udev-trigger-early.conf`
  - after udev trigger for DRM devices
  - frecon shows boot animation
  - frecon also spawns three agetty (developer consoles)
- `startup.conf` is triggered after `pre-startup.conf`
  - it invokes `chromeos_startup` to mount
- `boot-services.conf` is triggered after `startup.conf`
  - it is a milestone that many depend on
  - `start on starting boot-services`
    - `arc-system-mount.conf` invokes `arc-setup` to mount arc images
    - `auditd.conf` starts auditd
    - `bio_crypto_init.conf`
    - `cgroups.conf` mounts /sys/kernel/cgroup
    - `crash-reporter.conf`
    - `cros_configfs.conf` mounts `/usr/share/chromeos-config/configfs.img` to
      `/run/chromeos-config`
    - `dbus.conf` starts dbus-daemon
    - `ext-pci-drivers-allowlist.conf`
    - `syslog.conf` starts `rsyslogd`
    - `install-completed.conf` cleans up for first boot after update
    - `journald.conf` starts `systemd-journald`
    - `lockbox-cache.conf`
    - `metrics_library.conf`
    - `persist-crash-test.conf`
    - `preload-network.conf`
    - `swap.conf` enables swap on zram
    - `tracefs-init.conf` restores SELinux context for tracefs
    - `vpd-log.conf`
  - `start on started boot-services`
    - `bootlockboxd.conf` starts bootlockboxd
    - `cr50-update.conf` upates cr50 firmware
    - `cryptohomed.conf` starts cryptohomed
    - `factory.conf`
    - `failsafe-delay.conf`
    - `oobe_config_restore.conf`
    - `pca_agentd.conf` starts `pca_agentd`
    - `powerd.conf` starts powerd
    - `rt-limits.conf`
    - `tpm_managerd.conf` starts `tpm_managerd`
    - `trunksd.conf` starts `trunksd`
    - `wpasupplicant.conf` starts `wpa_supplicant`
    - `ui.conf` invokes `session_manager`
- `network-services.conf` is triggered after `boot-services.conf`
  - triggers `iptables.conf` to set up firewall
  - triggers `regulatory-domain.conf` to invoke `iw reg set`
  - triggers `shill.conf` to start shill (network-manager-like)
- `ui.conf` is triggered after `boot-services.conf`
- `boot-complete.conf` is triggered by `login-prompt-visible`
  - `login-prompt-visible` is an event sent by ui's `session_manager` after
    login is displayed
- `system-services.conf` is triggered by `boot-complete.conf`
  - `start on starting system-services`
    - `apk-cache-cleaner.conf`
    - `avahi.conf`
    - `conntrackd.conf`
    - `cras.conf`
    - `crash-sender.conf`
    - `cros-camera-algo.conf`
    - `cros-camera.conf`
    - `mipicam-device-added`
    - `cros-disks.conf`
    - `cros_healthd.conf`
    - `cryptohome-update-userdataauth.conf`
    - `dlcservice.conf`
    - `failsafe.conf`
    - `imageloader-init.conf`
    - `log-rotate.conf`
    - `modemmanager.conf`
    - `mtpd.conf`
    - `oemconfig.conf`
    - `patchpanel.conf`
    - `permission_broker.conf`
    - `preload-network.conf`
    - `update-engine.conf`
    - `upstart-socket-bridge.conf`
  - `start on started system-services`
    - `anomaly-detector.conf`
    - `biod.conf`
    - `bluetoothd.conf`
    - `bootchart.conf`
    - `btdispatch.conf`
    - `check-rw-vpd.conf`
    - `cleanup-shutdown-logs.conf`
    - `cpufreq.conf`
    - `cros-machine-id-regen-periodic.conf`
    - `crx-import.conf`
    - `displaylink-driver.conf`
    - `ec-report-tpsreset.conf`
    - `ecloggers.conf`
    - `mount-encrypted.conf`
    - `preload-network.conf`
    - `report-power-metrics.conf`
    - `send-boot-metrics.conf`
    - `send-boot-mode.conf`
    - `send-disk-metrics.conf`
    - `send-kernel-errors.conf`
    - `send-mount-encrypted-metrics.conf`
    - `send-powerwash-count.conf`
    - `send-reclamation-metrics.conf`
    - `send-recovery-metrics.conf`
    - `tlsdated.conf`
    - `trim.conf`
    - `u2fd.conf`
    - `ui-collect-machine-info.conf`
    - `uinput.conf`

## `chromeos_startup`

- a script invoked early by init
- mount debugfs, devpts, /dev/shm, configfs
- source `/usr/sbin/write_gpt.sh` and get partition layout
- mount `PARTITION_NUM_STATE` to `/mnt/stateful_partition`
- mount `PARTITION_NUM_OEM` to `/usr/share/oem`
- create common stateful directores
  - home, home/chronos, home/root, home/user
  - unencrypted, unencrypted/cache, unencrypted/preserve
- bind mount /mnt/stateful/home to /home
- `do_mount_var_and_home_chrono` from `/usr/share/cros/startup_utils.sh`
  - it invokes `mount-encrypted` to mount `/mnt/stateful/encrypted`
  - according to `/var/log/mount-encrypted.log`,
    - sets up a loop device for `/mnt/stateful_partition/encrypted.block`
    - sets up `/dev/mapper/encstateful` using `dm-crypt` from the loop
      device
    - mount dm device as ext4 to `/mnt/stateful_partition/encrypted`
    - bind mount encrypted var to /var
    - bind mount encrypted chronos to /home/chronos
- bind-mount /run to /var/run
- bind-mount /run/lock to /var/lock
- bind-mount /run/daemon-store/\* to self
- mount tmpfs to /media with `--make-shared`
- `dev_mount_packages` from `/usr/share/cros/dev_utils.sh`
  - dev only
  - bind mount `/mnt/stateful_partition/dev_image` to /usr/local
  - bind mount `/mnt/stateful_partition/var_overlay/*` to `/var/*`

## `session_manager`

- TODO

## `cryptohomed`

- note that `/home` is a bind mount of `/mnt/stateful/home`
  - `/home/.shadow` is just `/mnt/stateful/home/.shadow`
    - it is encrypted by ext4 encryption
  - but `/home/.shadow/chronos` is `/mnt/stateful_partition/encrypted/chronos`
    - it is encrypted by dm-crypt
- when cryptohomed is given username/password on user login,
  - it hashes the username to get an hash
  - `/home/.shadow/<salted_hash_of_username>` is where the user data are
    - `master.0` is the encrypted vault keyset (EVK) that can be descrypted by
      password to get VK
    - `mount` is an encrypted ext4 directory and can be descrypted by VK
  - it then bind mounts various user directories
    - `/home/.shadow/<hash>/mount/user` to `/home/user/<hash>` 
      - and to `/home/chronos/u-<hash>` for multi-user support
      - and to `/home/chronos/user` for backward compatibility
    - `/home/.shadow/<hash>/mount/root` to `/home/root/<hash>` 
  - for a new username/password, cryptohomed creates `/home/.shadow/<hash>`
    and everything underneath
- `/home/.shadow/<hash>/mount` uses ext4 encryption
  - file names and contents are encrypted
  - `e4crypt add_key` to add the key to the user keyring
  - everything is automatically decrypted on demand
  - it used to use stacked ecryptfs
- what are the differences between the three user directories?
  - `/home/user/<hash>` is for Chrome user data?
  - `/home/root/<hash>` is for system service user data?
  - `/home/chronos/u-<hash>` is for multi-user user data?

## Sandboxing

- `stop ui` first to look at system services
- minijail is wildly used by upstart confs
  - the only unused namespace type is user
- `lsns -t mnt`
  - `systemd-journald`
  - `rsyslogd`
  - `pca_agentd`
  - `bootlockboxd`
  - `cryptohome-proxy`
  - `cros_camera_service`
  - `cros_healthd`
  - `anomaly_detector`
  - `u2fd`
  - `mtpd`
  - `avahi-daemon`
  - `cras`
  - `conntrackd`
  - `dlcservice`
  - `cros_camera_algo`
  - `sslh-fork`
  - `btdispatch`
  - `tlsdate`
  - `metrics_daemon`
  - `memd`
