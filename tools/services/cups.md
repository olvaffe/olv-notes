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

## CUPS v2

- the current version, but being deprecated
- <https://github.com/OpenPrinting/cups>
  - backends: `dnssd`, `http(s)`, `ipp(s)`, `snmp`, `socket`, `usb`
  - filters: `commandtops`, `gziptoany`, `pstops`, `rasterto*`
  - bsd printing system compat
    - `lpc` configs queues
    - `lprm` cancels jobs
    - `lpq` queries queues
    - `lpr` submits jobs
  - sysv printing system compat
    - `lpadmin` configs queues
    - `cancel` cancels jobs
    - `lpmove` moves jobs
    - `lpstat` queries queues
    - `lp` submits jobs
  - sysv-inspired commands
    - `cupsaccept` accepts jobs
    - `cupsreject` rejects jobs
    - `cupsenable` enables queues
    - `cupsdisable`  disables queues
  - cups commands
    - `cupsctl` configs `/etc/cups/cupsd.conf`
    - `cupsd` implments IPP 2.1, most of IPP Everywhere, and a webadmin
      interface
    - `cupsfilter` converts files using filters
    - `cupstestppd` validates ppd files
    - `lpinfo` lists available drivers (`-m`) and available devices (`-v`)
    - `lpoptions` configs printers
  - ipp commands
    - `ippeveprinter` simulates an IPP Everywhere printer
      - this is the origin of PAPPL
    - `ippfind`
    - `ipptool`
  - ppd commands
    - `ppdc`
    - `ppdhtml`
    - `ppdi`
    - `ppdmerge`
    - `ppdpo`
- <https://github.com/OpenPrinting/cups-filters>
  - cups components that apple did not need
  - backends: `beh`, `driverless`, `parallel`, `serial`
  - drivers: `driverless`
  - filters: many, but the relevant ones are `pdftopdf` and `pdftoraster`
    - that is, modern cups takes pdfs and outputs one of the standard PDLs
      using `pdftopdf`, or `pdftoraster` followed by `rastertopwg`
- <https://github.com/OpenPrinting/cups-browsed>
  - `cups-browsed` sets up queues for all discovered printers automatically
  - `/etc/cups/cups-browsed.conf`
    - `CreateIPPPrinterQueues`
      - `All` is default and create queues for all discovered printers
      - `Driverless` creates queues for discovered printers that are capable
        of AirPrint or IPP Everywhere
    - `UseCUPSGeneratedPPDs`
      - `Yes` uses cups to generate ppds for created queues
      - `No` is the default and uses cups-filter to generate ppds for
        created queues
  - cups has a ppd generator
    - when creating a queue manually with `lpadmin -m everywhere`, cups
      generates a ppd for the queue by querying the IPP-Everywhere-capable
      printer
    - this should not be used anymore
  - cups-filter also has a ppd generator
    - `lpadmin -m driverless`
    - it works better than the one built into cups
    - `driverless` is also a standalone tool

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
- Other Libraries
  - <https://github.com/OpenPrinting/libcupsfilters> extracted from
    cups-filters for filtering (format conversions)
  - <https://github.com/OpenPrinting/libppd> extracted from cups for legacy
    PPD support

## Legacy Canon Printers

- ufrii, currently v6.00
  - mixed open/proprietary driver
    - UFR II stands for Ultra Fast Rendering v2
    - arch aur `cnrdrvcups-lb`
  - `/usr/lib/cups/backend/cnusbufr2` is the usb backend to communicate with
    the printer
  - `/usr/share/cups/model/*.ppd` describes the supported printers
  - `/usr/lib/cups/filter/rastertoufr2` converts cups raster format to UFR2,
    LIPSLX, UFR2 LT, or CARPS2 format
    - it forks off `/usr/bin/cnrsdrvufr2` to do the real work
  - `/usr/lib/cups/filter/pdftocpca` converts pdf to CPCA (Common Peripheral
    Controlling Architecture) format
    - it forks off `/usr/bin/cnpdfdrv` to do the real work
    - it does not appear to be used
  - essential package contents
    - `/usr/lib/cups/filter/rastertoufr2`
      - `/usr/bin/cnrsdrvufr2`
        - `/usr/lib/libufr2filterr.so.1.0.0`
        - `/usr/lib/libcaepcmufr2.so.1.0`
    - `/usr/lib/libcanonufr2r.so.1.0.0` is specified in `CN_PdlWrapper_PdlPath`
      - `/usr/lib/libcanon_slimufr2.so.1.0.0`
      - `/usr/bin/cnjbigufr2` compresses data using `jbigkit`
      - `/usr/bin/cnpkmoduleufr2r`
    - `/usr/share/caepcm`
    - `/usr/share/cups`
  - re-package deb
    - `dpkg-deb -R cnrdrvcups-ufr2-us_6.00-1.02_amd64.deb repack`
    - edit `repack/DEBIAN/control` to remove `cups-bsd` and `libgtk-3-0`
    - remove `repack/DEBIAN/post*`
    - `dpkg-deb -b repack repack.deb`
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

## Add a Local Printer

- `ID 04a9:2759 Canon, Inc. MF3010` is a legacy usb printer
  - `lpinfo -v` lists available devices
    - `direct cnusbufr2:/dev/usb/lp0`
    - `direct usb://Canon/MF3010?serial=foo&interface=1`
  - `lpinfo -m | grep 3010` lists available models (drivers)
    - `CNRCUPSMF3010ZS.ppd Canon MF3010`
  - `lpadmin -p mf3010 -E -v 'usb://Canon/MF3010?serial=foo&interface=1' -m 'CNRCUPSMF3010ZS.ppd'`
    adds a queue
    - `-p mf3010` adds queue `mf3010`
    - `-E` enables the queue and accepts jobs
    - `-v` specifies the device to use
    - `-m` specifies the driver to use
      - cups v3 is IPP Everywhere-only and this is deprecated
      - will need to use pappl to simulate an IPP Everywhere printer
  - `lpstat -t` shows queue stats
    - `scheduler is running` indicates cupsd is running
    - `no system default destination` indicates no default queue
      - `lpadmin -d mf3010` to set as default
    - `device for mf3010: usb://Canon/MF3010?serial=foo&interface=1` indicates
      queue `mf3010` uses device `usb://Canon/MF3010?serial=foo&interface=1`
    - `mf3010 accepting requests since ...` indicates the queue accepts jobs
      - `cupsreject mf3010` to reject
      - `cupsaccept mf3010` to accept
    - `printer mf3010 is idle.  enabled since ...` indicates the queue is
      idle and is enabled
      - `cupsdisable mf3010` to disable
      - `cupsenable mf3010` to enable
  - `lp -d mf3010 test.txt` prints a text file
- debug filtering
  - `cupsfilter -p /usr/share/ppd/cupsfilters/Generic-PDF_Printer-PDF.ppd test.txt > test.pdf`
    runs the filters as if we are printing `test.txt` to `Generic-PDF_Printer-PDF.ppd`
  - the input mime is auto-detected to be `text/plain`
  - the output mime is specified in ppd to be `application/pdf`
  - `universal` filter accepts all mimes and picks `texttopdf` filter to do
    the real work
  - `pdftopdf` filter uses qpdf to simplify the pdf
    - printers do not implement every pdf feature

## Share a Local Printer

- `lpadmin -p mf3010 -o printer-is-shared=true` marks `mf3010` shared
  - this updates `/etc/cups/printers.conf`
- `cupsctl --share-printers` makes `cupsd` accept remote requests
  - this updates `/etc/cups/cupsd.conf`
  - `Listen localhost:631` becomes `Port 631`
  - `Browsing On` advertises shared queues using avahi
  - `<Location />Order allow,deny  Allow @LOCAL</Location>`
    - all http requests starting with `/` are denied by default, but are
      allowed if coming from the local subnet
- `ippfind` finds the shared queue
  - `ipp://foo.local:631/printers/mf3010`
  - make sure avahi is running
  - might need to restart avahi and cupsd
  - does not seem to be too reliable
- to use the shared queue, on the remote machine,
  - install `cups-browsed` to automate queue creation
  - `lpadmin -p test -E -v 'ipp://foo.local:631/printers/mf3010' -m 'driverless:ipp://foo.local:631/printers/mf3010'`
    creates the queue manually
  - `driverless 'ipp://foo.local:631/printers/mf3010'` to see the generated
    ppd

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
