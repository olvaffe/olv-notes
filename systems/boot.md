Linux Boot Process
==================

## SysVinit (`/sbin/init`)

- PID 1
- it parses `/etc/inittab`
  - usually, it runs scripts in `/etc/rcS.d` and `/etc/rc2.d`.  Then runs
    `getty` for `tty[1-6]`
- `rcS.d`
  - `S01mountkernfs.sh` mounts `/run`, `/run/lock`, `/run/shm`, `/proc`, and
    `/sys` first.
  - `S02udev` makes sure `/dev` is mounted as (dev)tmpfs and starts udev
  - `S03mountdevsubfs.sh` mounts `/dev/pts`
  - `S05keyboard-setup` runs `setupcon` to set up kernel keymap
    - It is a script that runs `loadkeys`
    - This is an early setup for, say, checkroot failure interaction
  - `S07checkroot.sh` remounts `/` according to `/etc/fstab`
  - `S08hwclock.sh` sets kernel time from RTC
  - `S08kmod` load modules listed in `/etc/modules`
  - `S10mountall.sh` mounts all fs listed in `/etc/fstab`
  - `S13procps` runs `sysctl` for settings listed in `/etc/sysctl.conf`
  - `S15networking` runs `ifup`, which reads `/etc/network/interfaces`
  - `S16rpcbind` runs `rpcbind` to convert RPC prog number to port number.  It
    is used by services such as NFS or NIS.
  - `S18kbd` and `S19console-setup` run `setupcon`
  - `S20alsa-utils` calls `alsactl` to restoure alsa state
  - `S20fuse` mounts `/sys/fs/fuse/connections` as `fusectl`
- `rc2.d`
  - `S01acpi-fakekey` starts `acpi_fakekeyd`
    - it opens a FIFO at `/var/run/acpi_fakekey` to accept input from
      `acpi_fakekey`
    - it uses uinput to generate kernel key events
  - `S01binfmt-support` updates kernel binfmt database
  - `S01rsyslog` starts syslogd
  - `S02acpid` starts acpid.  It is a kernel ACPI event multiplexer.  It can
    also respond to ACPI events using actions defined in `/etc/acpi/events`
  - `S02dbus` starts system-wise dbus
  - `S02pulseaudio` starts system-wise pulseaudio.  Usually disabled.
  - `S02ssh` starts sshd
  - `S03avahi-daemon` starts avahi for mDNS/DNS-SD
  - `S03bluetooth` starts bluetoothd
  - `S03cpufrequtils` sets the CPU governor to ondemand
  - `S03network-manager` starts NetworkManager
  - `S04openvpn` starts openvpn
  - `S05cups` starts cups
  - `S05gdm3` starts gdm3

## udev

- `/lib/udev/rules.d`

## dbus

- `/usr/share/dbus-1/system-services`
- `/usr/share/dbus-1/services`

## GDM3

- see `gdm.md`
- The session is started by `/etc/gdm/Xsession <name>`
  - it sources `/etc/X11/Xsession.d`
  - it the ends, it exec() `ssh-agent dbus-launch x-session-manager`
    - or, `~/.xsession` if the file exists
- The default session runs `/etc/gdm/Xsession "default"`
  - `im-config_launch` starts the input method
    - it sources
      - `/usr/share/im-config/xinputrc.common`
      - `/etc/default/im-config`
      - `~/.xinputrc` or `/etc/X11/xinit/xinputrc`
    - usually it starts `00_default.rc`, which starts the input method set in
      `/etc/default/im-config`, which is usually `01_auto.rc`
- In `on_session_exited`, `PostSession/Default` is run

## Desktop

- `gnome-session`
  - `get_session_keyfile()`
    - it runs `IsRunnableHelper` to see if the session is runnable
    - for each provider listed in `RequiredProviders`, it checks if the
      app `DefaultProvider-<provider>` exists.  If not, the session is not
      runnable
    - for each app listed in `RequiredComponents`, it checks if the app is
      available.
    - if the session is not runnable, `FallbackSession` is used.
  - `load_standard_apps()`
    - call `gsm_manager_add_required_app()` on all `RequiredComponents`
    - call `gsm_manager_add_autostart_app()` for all autostart apps
    - call `gsm_manager_add_required_app()` on all `RequiredProviders`
  - when started with the session defined by `/usr/share/gnome-session/sessions/gnome.session`
    - which usually runs `gnome-shell` and `gnome-settings-daemon`
  - it also runs all apps defined in
    - `~/.config/autostart`
    - `/usr/share/gnome/autostart`
    - `/etc/xdg/autostart`
  - it provides a dbus service, `org.gnome.SessionManager`.  gnome-shell talks
    to the service to logout the session.
- Manually running `dbus-launch Xorg; gnome-settings-daemon` shows that
  gnome-settings-daemon starts
  - polkitd (for limiting power management)
  - upowerd (for doing low-level power management)
  - console-kit-daemon
- Manually running `gnome-shell` shows that it starts
  - accounts-daemon (used by g-c-c accounts)
  - gsd-printer (used by g-c-c printer)
  - packagekitd (used by g-c-c info)
  - colord (used by g-c-c color)
  - goa-daemon (used by g-c-c online-accounts)
  - gnome-shell-calendar-server
  - e-addressbook-factory (used by g-s calendar server)
  - mission-control-5 (used by g-s IM)
  - gconfd-2
  - dconf-service
  - gvfsd
- `gnome-settings-daemon`
  - it has a plugin-based design
  - it usually works by listening to various GSettings keys.  When the value
    of a key is changed (e.g., the background image), it responds (e.g., set
    the new background image)
  - for some other plugins, they listen to dbus signals (e.g., cups new job
    event) and/or export their functionalities (e.g., xrandr) on dbus.
  - xsettings plugin
    - it reads configurations from gsettings `org.gnome.desktop.interface` and
      etc and export them through XSETTINGS
      - e.g., `gsettings get org.gnome.desktop.interface gtk-theme`
    - libgtk+ and etc get the configurations from XSETTINGS
  - background plugin
  - power plugin
  - many others
