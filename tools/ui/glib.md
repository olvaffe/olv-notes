GLib
====

## Overview

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
