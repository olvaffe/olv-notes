GTK+
====

## Overview

- <https://gitlab.gnome.org/GNOME/gtk>

## GDK

- GDK has several backends
  - broadway, for html5
  - macos
  - wayland
  - win32
  - x11
- it also supports several renderers
  - opengl
  - vulkan
  - cairo
    - accurate, high-quality, but no gpu acceleration
    - kind of in maintenance mode

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

## IM

GDK: don't XFilterEvent key press/release

gtk_im_context --- gtk_im_context_simple
               \-- gtk_im_multicontext -- gtk_im_context_XXX in module form

gtk_im_context_simple: simple input method for Europe
gtk_im_multicontext: proxy for other im context
XIM im module: XFilterEvent

gtk_im_multicontext_{new,set_client_window,filter_keypress,focus_in,get_preedit_string,set_cursor_position}

## GObject Introspection

- <https://gitlab.gnome.org/GNOME/gobject-introspection>
- to ease creations of language bindings
- `g-ir-scanner` scans the source files to create GIR XML format
  - other than scanning the source files, it also build an executable to dump
    GTypes of the library.  The binary can dump the properties and signals of
    the library's GTypes.
  - run `g-ir-scanner` with `GI_SCANNER_DEBUG=save-temps` to see the source
    file of the executable files
- `g-ir-compiler` compiles GIR XML files into typelibs
- `libgirepository` is a C library for accessing typelib
- Support GObject Introspection
  - add `GOBJECT_INTROSPECTION_CHECK` to configure.ac
  - add these to Makefile.am
    - `include $(INTROSPECTION_MAKEFILE)`
      - `/usr/share/gobject-introspection-1.0/Makefile.introspection`
    - `INTROSPECTION_GIRS = List of GIRs to be generated`
    - `INTROSPECTION_SCANNER_ARGS = Additional args for g-ir-scanner`
    - `INTROSPECTION_COMPILER_ARGS = Additional args for g-ir-compiler`

## GLib

- <https://gitlab.gnome.org/GNOME/glib>

## GSettings

- a replacement for gconf built into glib
- have multiple backends for its storage
  - dconf being one of them

## gvfs and gio

- a replacement for gnome-vfs built into glib 
- gio is the API, and gvfs is the implementation (with daemons, and etc.)

## GDBus

- does not use libdbus
- use gio as the transport layer

## GVariant

## GObject

signals are synchronous.
g_object_unref (destroy, finalize, whatever) is asynchronous.


PROPERTY
---------------

xxx_set_property
{
	xxx_ooo_set();
}

xxx_get_property
{
	return priv->ooo;
}

xxx_ooo_set
{
	blah;
	priv->ooo = ooo;

	g_signal_emit(OOO_CHANGED);
}

xxx_ooo_get
{
	return priv->ooo;
}

Having ooo as property, we have

- a generic way to set ooo -> themeable!
- 


---------------
g_signal_new creates a class_closure when there is class_offset, which has meta_marshal (g_type_class_meta_marshal).
The c_marshaller passed to g_signal_new is stored in SignalNode and also the marshal for class_closure.

On g_signal_connect, a cclosure is created for the callback function and stored
in Handler.  SignalNode->c_marshaller is used as the marshal for the cclosure.


---------
instance allocated -> xxx_ooo_init called ->
	constructor called with construct time properties -> set normal properties
