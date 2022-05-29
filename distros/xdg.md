XDG Specs
=========

## Base Directory Specification

- <https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html>
- `XDG_DATA_HOME`
  - `$HOME/.local/share`
- `XDG_CONFIG_HOME`
  - `$HOME/.config`
- `XDG_STATE_HOME`
  - `$HOME/.local/state`
- `XDG_CACHE_HOME`
  - `$HOME/.cache`
- `XDG_DATA_DIRS`
  - `/usr/local/share/:/usr/share/`
- `XDG_CONFIG_DIRS`
  - `/etc/xdg`
- `XDG_RUNTIME_DIR`
  - not set

## `pam_systemd`

- `man pam_systemd`
  - `XDG_SESSION_ID`
  - `XDG_RUNTIME_DIR`
  - `XDG_SESSION_TYPE`
  - `XDG_SESSION_CLASS`
  - `XDG_SESSION_DESKTOP`
  - `XDG_SEAT`
  - `XDG_VTNR`

## Desktop Entry Specification

- <https://specifications.freedesktop.org/desktop-entry-spec/latest/>
- `XDG_CURRENT_DESKTOP`
  - If `$XDG_CURRENT_DESKTOP` is set then it contains a colon-separated list
    of strings. In order, each string is considered. If a matching entry is
    found in `OnlyShowIn` then the desktop file is shown. If an entry is found
    in `NotShowIn` then the desktop file is not shown. If none of the strings
    match then the default action is taken (as above).

## Recent File Storage Specification

- uses `~/.recently-used`
- gnome uses `$XDG_DATA_HOME/recently-used.xbel`

## Thumbnail Managing Standard

- uses `~/.thumbnails`

## Trash specification

- uses `$XDG_DATA_HOME/Trash`

## Others

- Flash player
  - it creates `~/.adobe` and `~/.macromedia`
- Nautilus
  - it uses `.gtk-bookmarks` for bookmarks
