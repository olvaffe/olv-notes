GLib
====

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
