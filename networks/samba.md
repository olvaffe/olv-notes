Samba
=====

## Overview

- an implementation of SMB protocol
- windows and macs both use SMB
  - macs used to use AFP, Apple Filing Protocol
  - there is also netatalk, <https://netatalk.sourceforge.io/>, to implement
    AFP
  - but macs use smb now

## Old

- `smbclient -U olv.wu -W azwave //tpfs01/data`
- Vista sees:
  - accounts:
    - Vista Home Basic must use current username/password (token?) to login samba
    - that means there must be an account with the same password on the server
    - and it cannot login with another name
  - `//pinky/PRINTERS`
    - remote printer server
    - spoolers' names are shown
    - allow remote control, with uniform ui as controlling local printer server
    - require proper permissions to modify
    - they could be shadowed by local copies!
  - `//pinky/some-printer`:
    - a remote spooler shared
    - local printer server could use it, after installing a local copy
  - local copy of `//pinky/some-printer`:
    - is a real printer.  it needs a driver.
    - if the remote one has a driver, it will be copied and used
    - otherwise, CD is needed for installation
    - the two cases are very different
    - in the former case, local copy is like a hard-link to the remote one
    - in the former case, local copy shadows the remote one
    - and even installed, if no "user client driver = yes", spooler queue could not be shown
- samba `use client driver` option:
  - printers with drivers (setdriver'ed): NO
  - printers without drivers: YES
  - clients could install drivers locally or upload to samba
  - if clients decide to install drivers locally, should be YES, otherwise spooler queue could not get the status
  - if clients want to upload, must be NO.  But clients must be sure to have SePrintOperatorPrivilege or be printer admin
  - Also, clients must know HOW to upload from windows
  - with it set to YES, it is not possible to associate a spooler with a driver
- net
  - `net rpc rights grant some-user SePrintOperatorPrivilege`
- `cupsaddsmb`
  - see net secion
  - user must be in write list
  - NO "use client driver"
  - permissions to /var/lib/samba/printers
  - put drivers under /usr/share/cups/drivers
  - run it
- `rpcclient`
  - enumdrivers         Enumerate installed printer drivers
  - enumprinters        Enumerate printers
  - getdriver           Get print driver information
  - getdriverdir        Get print driver upload directory
  - adddriver
  - deldriver
  - setdriver: no "use client driver", otherwise access denied
- Internals
  - `get_iconv_convenience` returns an ic, which maps `CH_DOS` to ASCII and
    `CH_UNIX` to UTF-8.  `pull_ascii_talloc` uses ic to convert from `CH_DOS`
    to `CH_UNIX`.  `srvstr_pull_req_talloc`, depending on `smb_flags2`, calls
    `pull_ucs2_base_talloc` (`CH_UTF16LE` to `CH_UNIX`) or
    `pull_ascii_base_talloc` (`CH_DOS` to `CH_UNIX`).
  - Internally, samba uses `CH_UNIX`.  Nothing special needed to to strcmp.
    But for strlen or strchr, a special multibytes version is used.
  - `smb_messages` in `process.c` gives how to dispatch all smb commands.
    After a request is dispatched, `srvstr_pull_req_talloc` is called to pull
    string out of the request and convert it from `CH_UTF16LE` or `CH_DOS` to
    `CH_UNIX`.
  - First request is `reply_negprot`, which decides remote arch (`ra_arch`),
    `reload_services`, decides protocol (`remote_proto`), and reply with
    protocol-specific reply function, to negociate the capabilities
  - Several `reply_sesssetup_and_X` follows, which give the username/password
    or anonymous
  - Then `reply_tcon_and_X`, which calls `make_connection` to requested share
  - Then `reply_ntcreate_and_X`, to create `\srvsvc`.
  - Then `reply_trans` (`rpc:Bind`, `rpc:NetShareEnumAll`), which uses
    `ndr_gen` (e.g. `ndr_push_srvsvc_NetShareEnumAll`) and finally
    `reply_close`.
