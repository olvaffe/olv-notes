XDG Specs
=========

## Base Directory Specification

- `XDG_CACHE_HOME`
  - `~/.cache`
- `XDG_CONFIG_HOME`
  - `~/.config`
- `XDG_DATA_HOME`
  - `~/.local/share`
- `XDG_DATA_DIRS`
  - `/usr/local/share/:/usr/share/`
- `XDG_CONFIG_DIRS`
  - `/etc/xdg`
- `XDG_RUNTIME_DIR`
  - not set

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
