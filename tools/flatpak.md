Flatpak
=======

## Usage

- `pacman -S flatpak`
  - when prompted for portal, select `xdg-desktop-portal-wlr` for wlroots
- `flatpak search <app>` to search
- `flatpak install <appid>` to install
  - apps are installed to `/var/lib/flatpak`
- `flatpak run <appid>` to run

## `xdg-desktop-portal`

- <https://flatpak.github.io/xdg-desktop-portal/>
  - provides dbus interfaces known as portals under the well-known name
    `org.freedesktop.portal.Desktop` and object path
    `/org/freedesktop/portal/desktop`
  - why
    - provide sandboxed apps (e.g., flatpak apps) access to host desktop
      features
    - some features also turn out to be useful for non-sandboxed apps (e.g.,
      screencast)
  - `org.freedesktop.portal.Camera` interface provides camera access
  - `org.freedesktop.portal.ScreenCast` interface provides screencast
    capability
- config
  - `man portals.conf`
  - `$XDG_DATA_DIRS/xdg-desktop-portal/$XDG_CURRENT_DESKTOP-portals.conf`
  - `$XDG_DATA_DIRS/xdg-desktop-portal/portals.conf`
- since `xdg-desktop-portal` is usually started as a systemd user session
  service
  - `systemctl --user import-environment XDG_CURRENT_DESKTOP` to propagate the
    env
    - `dbus-update-activation-environment --systemd XDG_CURRENT_DESKTOP`
  - `systemctl --user restart xdg-desktop-portal` to restart
  - `journalctl --user -u xdg-desktop-portal` to see the log
