CUPS
====

## Modern Printers and Standards

- modern printer specs
  - sleep power: 0.5W
  - network printing
    - ethernet/wifi
    - mDNS/DNS-SD
    - IPP
  - usb printing
    - USB Printer Class
    - IPP
  - certifications
    - IPP Everywhere
    - Mopria
    - AirPrint
- standard bodies
  - PWG, The Printer Working Group, by IEEE
    - <https://www.pwg.org/>
    - PWG 5100.x: IPP, IPP Everyhwere
    - PWG 5102.x: PWG Raster Format
  - Mopria, by printer vendors
    - <https://mopria.org/>
  - AirPrint, by Apple
- standard PDLs (page description languages):
  - <https://openprinting.github.io/driverless/01-standards-and-their-pdls/>
  - PDF: successor of PostScript
  - PWG Raster: based on cusp's raster
  - PCLM: raster-only subset of PDF
  - Apple Raster: based on cusp's raster
  - JPEG
- how does a modern printer work?
  - <https://openprinting.github.io/driverless/02-workflow/>
  - discovery: mDNS/DNS-SD
  - communication: IPP
  - PDLs: varies

## CUPS v3

- not ready yet
- <https://openprinting.github.io/current/>
- <https://github.com/OpenPrinting/libcups> implements IPP and standard PDLs
  - it takes pdf as input and can output pdf/pwg/etc
- <https://github.com/OpenPrinting/cups-local> is a local print daemon
  - it discovers and uses libcups to talk to IPP printers
  - apps talk to this local daemon
- <https://github.com/OpenPrinting/cups-sharing> is a remote print daemon
  - it discovers and uses libcups to talk to IPP printers
  - it provides remote access
  - apps can talk to this remote daemon as well
- non-IPP printers
  - <https://github.com/michaelrsweet/pappl> is a framework to emulate IPP
    printers
  - <https://github.com/OpenPrinting/hplip-printer-app> emulates IPP printers
    on top of HPLIP which supports many HP printers
  - <https://github.com/OpenPrinting/gutenprint-printer-app> emulates IPP
    printers on top of Gutenprint which supports many Canon/Epson printers
  - <https://github.com/OpenPrinting/ghostscript-printer-app> emulates IPP
    printers on top of GhostScript which supports a variety of printers
  - <https://github.com/OpenPrinting/ps-printer-app> emulates IPP printers on
    top of PostScript printers
  - <https://github.com/OpenPrinting/pappl-retrofit> emulates IPP printers on
    top of CUPS legacy printers

## Overview

- <https://openprinting.github.io/cups/>
  - a fork of apple cups for all operating systems

## Old

- cups
  - drivers come from ppds and `/usr/lib/cups/driver/*`
  - to enum devices, `lpinfo -v`
  - to enum driver, `lpinfo -m`
  - RAW: `lpadmin -p <name> -v <device> -E`
  - COOKED: `lpadmin -p <name> -v <device> -P <ppd> -E`
- `scheduler/dirsvc.c` to DNS-SD
- data formats
  - CUPS -> printer: CUPS will check if it can convert the data to a format
    that the printer understands.
  - client -> CUPS: client is supposed to send in a format CUPS understands.
    - CUPS only accepts files of types defined in /etc/cups/mime.types.  Edit
      also /etc/cups/mime.convs
  - CUPS with gs support: pdftops, pstoraster!
  - CUPS without gs support: Many formats are no longer supported.
- Best Practice for print server:
  - CUPS without gs support + Raw support + Windows client driver
  - On usb device inserted/removed, do something
  - Listen on public iface
  - Printers should be published for remote printing (samba is local)
