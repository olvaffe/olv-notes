D-Bus
=====

## Overview

- <https://www.freedesktop.org/wiki/Software/dbus/>
- <https://dbus.freedesktop.org/doc/dbus-specification.html> is the spec
- <https://gitlab.freedesktop.org/dbus/dbus> is the reference impl
  - `libdbus` is still in use
  - distros are replacing `dbus-daemon` by
    <https://github.com/bus1/dbus-broker>

## D-Bus Specification

- <https://dbus.freedesktop.org/doc/dbus-specification.html>
- Type System
  - fixed types
    - `BYTE` is `y`
    - `BOOLEAN` is `b`
    - `INT16` is `n`
    - `UINT16` is `q`
    - `INT32` is `i`
    - `UINT32` is `u`
    - `INT64` is `x`
    - `UINT64` is `t`
    - `DOUBLE` is `d`
    - `UNIX_FD` is `h`
  - string-like types
    - `STRING` is `s`
    - `OBJECT_PATH` is `o`
    - `SIGNATURE` is `g`
  - container types
    - `ARRAY` is `a`
    - `STRUCT` is `r`, but its signature is `()`
    - `VARIANT` is `v`
    - `DICT_ENTRY` is `e`, but its signature is `{}`
- Marshaling (Wire Format)
- Message Protocol
  - a message consists of a header and a body
  - the signature of the header is `yyyyuua(yv)`
    - 1st byte: endianness flag
    - 2nd byte: message type
      - `INVALID`, `METHOD_CALL`, `METHOD_RETURN`, `ERROR`, `SIGNAL`
    - 3nd byte: flags
      - `NO_REPLY_EXPECTED`, `NO_AUTO_START`, `ALLOW_INTERACTIVE_AUTHORIZATION`
    - 4th byte: major protocol version
    - 1st u32: body length
    - 2nd u32: seqno
    - zero or more header fields
      - `PATH`, the object path
      - `INTERFACE`, the interface name
      - `MEMBER`, the interface member
      - `ERROR_NAME`
      - `REPLY_SERIAL`, the message seqno this reply is for
      - `DESTINATION`, the destination
      - `SENDER`, this is set by the message bug and is trustworthy
      - `SIGNATURE`, signature of the body
      - `UNIX_FDS`, number of fds
- Authentication Protocol
- Server Addresses
- Transports
- Meta Transports
- UUIDs
- Standard Interfaces
  - `org.freedesktop.DBus.Peer`
  - `org.freedesktop.DBus.Introspectable`
  - `org.freedesktop.DBus.Properties`
  - `org.freedesktop.DBus.ObjectManager`
- Introspection Data Format
- Message Bus Specification
  - the message bus accepts connections from applications
    - each connection is assigned a unqiue name
    - a connection can also "own" a well-known name
  - the bus itself owns a special name, `org.freedesktop.DBus`, with object
    `/org/freedesktop/DBus` that implements interface `org.freedesktop.DBus`
  - a message may be unicast or broadcast depending on whether `DESTINATION`
    is set
    - method calls and relies are typically unicast
    - signals are typically broadcast
  - when a message is sent to an unowned well-known name, the bus might
    auto-start the app that will own the name
    - `/usr/share/dbus-1/system-services`
    - `/usr/share/dbus-1/services`
  - Well-known Message Bus Instances
    - system bus
      - the addr is specified by `DBUS_SYSTEM_BUS_ADDRESS`, or default to
        `/run/dbus/system_bus_socket`
    - login session bus
      - the addr is specified by `DBUS_SESSION_BUS_ADDRESS` and is typically
        `/run/user/<uid>/bus`
  - Message Bus Interface: `org.freedesktop.DBus`
  - Monitoring Interface: `org.freedesktop.DBus.Monitoring`
  - Debug Statistics Interface: `org.freedesktop.DBus.Debug.Stats`
  - Verbose Interface: `org.freedesktop.DBus.Verbose`

## `dbus-send`

- `dbus-send --print-reply --dest=org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus.ListNames`
- `sudo dbus-send --system --print-reply --dest=org.freedesktop.NetworkManager.dnsmasq / org.freedesktop.DBus.Introspectable.Introspect`
