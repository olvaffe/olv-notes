CUPS
====

## Overview

- <https://openprinting.github.io/cups/>
  - a fork of apple cups for all operating systems

## old

cups:
* drivers come from ppds and /usr/lib/cups/driver/*
* to enum devices, lpinfo -v
* to enum driver, lpinfo -m
* RAW: lpadmin -p <name> -v <device> -E
* COOKED: lpadmin -p <name> -v <device> -P <ppd> -E

scheduler/dirsvc.c to DNS-SD

CUPS -> printer: CUPS will check if it can convert the data to a format that
the printer understands.

client -> CUPS: client is supposed to send in a format CUPS understands.

Unable to convert file 0 to printable format for job?

CUPS only accepts files of types defined in /etc/cups/mime.types.  Edit also
/etc/cups/mime.convs

CUPS with gs support: pdftops, pstoraster!
CUPS without gs support: Many formats are no longer supported.

Best Practice for print server:
CUPS without gs support + Raw support + Windows client driver

On usb device inserted/removed, do something

Listen on public iface
Printers should be published for remote printing (samba is local)
