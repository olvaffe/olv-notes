Samba
=====

## Overview

- an implementation of SMB protocol
- windows and macs both use SMB
  - macs used to use AFP, Apple Filing Protocol
  - there is also netatalk, <https://netatalk.sourceforge.io/>, to implement
    AFP
  - but macs use smb now

## ksmbd

- the in-kernel server has been upstreamed
  - <https://git.samba.org/?p=ksmbd.git;a=summary>
- userspace daemon and tools
  - <https://github.com/cifsd-team/ksmbd-tools>
- it provides just file server and is not a full replacement for samba
- setup
  - `/etc/ksmbd/ksmbd.conf` defines shares
  - `ksmbd.adduser <name>` adds a user
    - it modifies `/etc/ksmbd/ksmbdpwd.db` which is plain text
  - `systemctl enable --now ksmbd` enables `ksmbd.mountd`
