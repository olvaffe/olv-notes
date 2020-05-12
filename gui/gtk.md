GTK+
====

## GtkSettings in 2.0

- `gtk_init()` calls `_gtk_rc_init()` to add `/etc/gtk-2.0/gtkrc` and
  `~/.gtkrc-2.0` as the default RC files
- There is a `GtkSettings` for each screen
  - it has a huge list of properties, such as `gtk-theme-name`,
    `gtk-double-click-time`, and etc.
  - As properties, they have hardcoded default values.
  - When XSETTINGS is available, these properties map to XSETTINGS directly
    through `gdk_screen_get_setting`
  - a `GtkRcContext` is associated with the settings.  When the settings is
    created, it calls `gtk_rc_reparse_all_for_settings()` to reparse resource
    files and `gtk_widget_reset_rc_styles()` all widgets
    - Other than gtkrc files, it also parses theme's rc file, using the theme
      name from the settings.
- RC files are parsed into `GtkRcStyle`
- Given a widget, `gtk_rc_get_style()` is called to get its `GtkStyle`
  - it creates a `GtkStyle` and applies all RC styles that match the widget on
    it
- When `gtk-theme-name` of GtkSettings is changed, `gtk_rc_settings_changed()`
  is notified, and it calls `gtk_rc_reparse_all_for_settings()` to make the
  new theme effective.

## Theming

- In GTK+ 3.0, `GtkStyle` is replaced by `GtkStyleContext`.  `GtkRcStyle` is
  replaced by `GtkCssProvider`
- Both `GtkSettings` and `GtkCssProvider` implement `GtkStyleProvider`.  They
  provide style information to `GtkStyleContext`
