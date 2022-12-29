XDG Specs
=========

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
