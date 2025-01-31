XDG Specs
=========

## XDG Base Directory Specification

- <https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html>
  - freedesktop.org was formerly known as X Desktop Group, XDG
- user-specific
  - `$XDG_CONFIG_HOME`, default to `$HOME/.config`
  - `$XDG_DATA_HOME`, default to `$HOME/.local/share`
  - `$XDG_STATE_HOME`, default to `$HOME/.local/state`
  - `$XDG_CACHE_HOME`, default to `$HOME/.cache`
  - `$XDG_RUNTIME_DIR`, no default
    - if exists, the directory must be owned by user and has access mode 0700
- system-wide
  - `$XDG_DATA_DIRS`, default to `/usr/local/share/:/usr/share/`
  - `$XDG_CONFIG_DIRS`, default to `/etc/xdg`
- when referenced from other specs,
  - `$XDG_DATA_DIRS/foo/bar` means
    - lookup should search `foo/bar` relative to all dirs listed in
      `$XDG_DATA_HOME` and `$XDG_DATA_DIRS`
    - installation should install to `$datadir/foo/bar`, with `$datadir`
      defaulting to `/usr/share`
  - `$XDG_CONFIG_DIRS/foo/bar` means
    - lookup should search `foo/bar` relative to all dirs listed in
      `$XDG_CONFIG_HOME` and `$XDG_CONFIG_DIRS`
    - installation should install to `$sysconfdir/xdg/foo/bar`, with
      `$sysconfdir` defaulting to `/etc`

## Desktop Entry Specification

- <https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html>
- desktop entry dirs are `$XDG_DATA_DIRS/applications`
- `XDG_CURRENT_DESKTOP`
  - If `$XDG_CURRENT_DESKTOP` is set then it contains a colon-separated list
    of strings. In order, each string is considered. If a matching entry is
    found in `OnlyShowIn` then the desktop file is shown. If an entry is found
    in `NotShowIn` then the desktop file is not shown. If none of the strings
    match then the default action is taken (as above).
  - `$XDG_CURRENT_DESKTOP` should have been set by the login manager,
    according to the value of the `DesktopNames` found in the session file.

## Desktop Menu Specification

- <https://specifications.freedesktop.org/menu-spec/menu-spec-latest.html>
- `$XDG_CONFIG_DIRS/menus/${XDG_MENU_PREFIX}applications.menu`
- `$XDG_CONFIG_DIRS/menus/applications-merged/`
- `$XDG_DATA_DIRS/applications/`
- `$XDG_DATA_DIRS/desktop-directories/`

## Desktop Application Autostart Specification

- <https://specifications.freedesktop.org/autostart-spec/autostart-spec-latest.html>
- autostart dirs are `$XDG_CONFIG_DIRS/autostart`
  - that means `~/.config/autostart` and `/etc/xdg/autostart` by default

## Shared MIME-info Database

- <https://specifications.freedesktop.org/shared-mime-info-spec/shared-mime-info-spec-latest.html>
- mime db dirs are `$XDG_DATA_DIRS/mime`
- `update-mime-database` scans `$XDG_DATA_DIRS/mime/packages` and generates
  - `aliases`
  - `generic-icons`
  - `globs`
  - `globs2`
  - `icons`
  - `magic`
  - `mime.cache`
  - `subclasses`
  - `XMLnamespaces`
  - `<media>/<subtype>.xml`

## MIME Types and Applications

- <https://specifications.freedesktop.org/mime-apps-spec/mime-apps-spec-latest.html>
- the lookup order is
  - `$XDG_CONFIG_HOME/$desktop-mimeapps.list`
  - `$XDG_CONFIG_HOME/mimeapps.list`
  - `$XDG_CONFIG_DIRS/$desktop-mimeapps.list`
  - `$XDG_CONFIG_DIRS/mimeapps.list`
  - `$XDG_DATA_HOME/applications/$desktop-mimeapps.list` (for compat)
  - `$XDG_DATA_HOME/applications/mimeapps.list` (for compat)
  - `$XDG_DATA_DIRS/applications/$desktop-mimeapps.list`
  - `$XDG_DATA_DIRS/applications/mimeapps.list`

## Icon Theme Specification

- <https://specifications.freedesktop.org/icon-theme-spec/icon-theme-spec-latest.html>
- icon theme dirs are, in order,
  - `$HOME/.icons` (for backwards compatibility)
  - `$XDG_DATA_DIRS/icons`, and
  - `/usr/share/pixmaps`

## Thumbnail Managing Standard

- <https://specifications.freedesktop.org/thumbnail-spec/thumbnail-spec-latest.html>
- the thumbnail dir is `$XDG_CACHE_HOME/thumbnails`

## Trash Specification

- <https://specifications.freedesktop.org/trash-spec/trashspec-latest.html>
- home trash dir is `$XDG_DATA_HOME/Trash`
- top dirs, if supported, are `$topdir/.Trash/$uid` or `$topdir/.Trash-$uid`
  - a `$topdir` is the mountpoint of a partition
  - this is to avoid copying

## Desktop Bookmark

- <https://www.freedesktop.org/wiki/Specifications/desktop-bookmark-spec/>
- `$XDG_USER_DATA/recently-used.xbel`
  - bookmarks to recently used resources
- `$XDG_USER_DATA/recent-applications.xbel`
  - bookmarks to recently used applications (to their desktop entries)
- `$XDG_USER_DATA/shortcuts.xbel`
  - bookmarks to user-defined folders

## Others

- <https://gitlab.freedesktop.org/xdg/desktop-file-utils>
  - `update-desktop-database` scans `*.desktop` and generates
    `$XDG_DATA_DIRS/applications/mimeinfo.cache`
- <https://cgit.freedesktop.org/xdg/xdg-user-dirs/>
  - `$XDG_CONFIG_DIRS/user-dirs.conf` is the config file
    - `enabled=`
    - `filename_encoding=`
  - `$XDG_CONFIG_DIRS/user-dirs.defaults` is the default names
    - `DESKTOP=Desktop`
    - `DOWNLOAD=Downloads`
    - `TEMPLATES=Templates`
    - `PUBLICSHARE=Public`
    - `DOCUMENTS=Documents`
    - `MUSIC=Music`
    - `PICTURES=Pictures`
    - `VIDEOS=Videos`
  - when `xdg-user-dirs-updates` runs for the first time,
    - it parses `user-dirs.defaults` for the default names
    - it translates the default names to localized names
    - it creates those dirs under `$HOME`
    - it writes the current names to `$XDG_CONFIG_HOME/user-dirs.dirs`
      - it is in `XDG_${NAME}_DIR="$HOME/<name>"` form and can be sourced by
        shell
    - it writes the current locale to `$XDG_CONFIG_HOME/user-dirs.locale`
      - this is to detect locale change later
  - when `xdg-user-dirs-updates` runs again, it parses
    `$XDG_CONFIG_HOME/user-dirs.dirs`
    - if there is `XDG_${NAME}_DIR=<path>` and `<path>` does not exist, it
      assumes the user does not want it anymore and changes `<path>` to
      `$HOME`
