CUPS
====

## Overview

- <https://openprinting.github.io/cups/>
  - a fork of apple cups for all operating systems

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

- <https://openprinting.github.io/current/>
  - not ready yet
  - only support IPP for printer discovery
  - only support IPP for printer communication
  - only support standard PDLs
    - PDF, PWG Raster, PCLm, Apple Raster
    - it takes pdf from client and converts to one of the standard PDLs
      supported by the IPP printer
- <https://github.com/OpenPrinting/libcups> implements IPP and standard PDLs
  - it takes pdf as input and can output pdf/pwg/etc
- <https://github.com/OpenPrinting/cups-local> is a local print daemon
  - it discovers and uses libcups to talk to IPP printers
  - apps talk to this local daemon
- <https://github.com/OpenPrinting/cups-sharing> is a remote print daemon
  - it discovers and uses libcups to talk to IPP printers
  - it provides remote access
  - apps can talk to this remote daemon as well
- <https://github.com/michaelrsweet/pappl> is a framework to emulate IPP
  printers on top of legacy drivers
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

## Legacy Canon Printers

- ufrii, currently v6.00
  - mixed open/proprietary driver
    - UFR II stands for Ultra Fast Rendering v2
  - `/usr/lib/cups/backend/cnusbufr2` is the usb backend to communicate with
    the printer
  - `/usr/share/cups/model/*.ppd` describes the supported printers
  - `/usr/lib/cups/filter/rastertoufr2` converts cups raster format to UFR2,
    LIPSLX, UFR2 LT, or CARPS2 format
    - it forks off `/usr/bin/cnrsdrvufr2` to do the real work
  - `/usr/lib/cups/filter/pdftocpca` converts pdf to CPCA format?
    - it forks off `/usr/bin/cnpdfdrv` to do the real work
    - it does not appear to be used
- cnijfilter2, currently v6.80
  - mixed open/proprietary driver
    - IJ stands for inkjet
    - supports Pixma inkjet printers
  - `/usr/lib/cups/backend/cnijbe2` is the usb/net backend to communicate with
    the printer
    - it forks off `/usr/bin/cnijlgmon3` to do the real work
  - `/usr/share/cups/model/*.ppd` describes the supported printers
  - `/usr/lib/cups/filter/cmdtocanonij2` converts cups cmds to IJ2 cmds for
    some printers
  - `/usr/lib/cups/filter/cmdtocanonij3` converts cups cmds to IJ3 cmds for
    some printers
  - `/usr/lib/cups/filter/rastertocanonij` converts cups raster format to ij
    format
    - it forks off `/usr/bin/tocnpwg` to do the real work
- <https://github.com/mounaiban/captdriver>
  - reverse-engineered
    - CAPT stands for Canon Advanced Printing Technology
    - supports LBP laser printers from 2010s?
    - there is also an official proprietary driver
  - `/usr/share/cups/model/*.ppd` describes the supported printers
  - `/usr/lib/cups/filter/rastertocapt` converts cups raster format to capt
    format
- <https://github.com/ondrej-zary/carps-cups>
  - reverse-engineered
    - CARPS stands for Canon Advanced Raster Printing System
    - supports some printers from 2000s?
  - `/usr/share/cups/usb/carps.usb-quirks` describes printer quirks to cups
    usb backend
  - `/usr/share/cups/drv/carps.drv` describes the supported drivers
    - `ppdc` can convert `.drv` to `.ppd`s
  - `/usr/lib/cups/filter/rastertocarps` converts cups raster format to carps
    format

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
