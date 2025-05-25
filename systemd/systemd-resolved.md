systemd-resolved
================

## systemd-resolved

- systemd-resolved is mainly a service that provides name resolutions to local
  apps
  - it synthesizes DNS resource records for these local names
    - the current hostname maps to the current ip addrs, or `127.0.0.2` if no
      current ip addr
    - `localhost`, `localhost.localdomain`, `.localhost` and
      `.localhost.localdomain` map to `127.0.0.1`
    - `_gateway` maps to the current default gateway addrs
    - `_outbound` maps to the current ip addr
    - `_localdnsstub` maps to `127.0.0.53`
    - `_localdnsproxy` maps to `127.0.0.54`
    - `/etc/hosts` are parsed
  - the name may be resolved locally, or resolved by upstream DNS, LLMNR, or
    mDNS
    - names matching the synthetic records are always resolved locally
    - single-label (no dots) non-synthesized names are resolved using LLMNR if
      enabled
    - single-label non-synthesized names are resolved using DNS with search
      domains
    - multi-label names with `.local` suffix are resolved using mDNS if
      enabled
    - multi-label names with other suffices are resolved using DNS
    - addr lookups (reverse lookups) are handled similar to multi-label names,
      with the exception of link-local addr lookups (`169.254.0.0/16`) which
      are resolved by LLMNR/mDNS if enabled
- note that systemd-resolved is also an LLMNR/mDNS responder
  - `man systemd.dnssd` documents how to announce a network service in mDNS
    - the service is assumed to be hosted on the local machine whose name is
      `<hostname>.local`
  - as far as I can see, it only responds to `<hostname>.local` resolution,
    but not others configured in `/etc/hosts`
    - this appears to be different from avahi and `/etc/avahi/hosts`
- it provides 3 interfaces for name resolutions
  - the native d-bus interface, `org.freedesktop.resolve1`
  - the posix `getaddrinfo`
    - `nsswitch.conf` must be configured to use `nss-resolve` provided by
      systemd-resolved
  - a local DNS stub listener on `127.0.0.53` and `127.0.0.54` of `lo`
    - this is for apps who use glibc resolver directly
    - `resolv.conf` must be configured to point to the local DNS stub listener
    - the `.54` one is a proxy, in that it passes DNS messages through
      unmodified mostly
- there are 4 options for `/etc/resolv.conf`
  - a symlink to `/run/systemd/resolve/stub-resolv.conf` (recommended)
    - it lists `127.0.0.53` as the only server and updates search domains
      dynamically
  - a symlink to `/usr/lib/systemd/resolv.conf`
    - it lists `127.0.0.53` as the only server with no search domains
  - a symlink to `/run/systemd/resolve/resolv.conf`
    - it lists the upstream DNS server and allows systemd-resolved to be
      bypassed
  - externally-maintained 
    - systemd-resolved becomes a consumer and parses the file for upstream DNS
      server
- when systemd-networkd receives DNS servers from the DHCP server, it tells
  `systemd-resolved` to use them as the upstream DNS servers
  - when systemd-networkd is not used, programs such as `dhcpcd` invokes
    `resolvconf` to manage `/etc/resolv.conf`
- `resolvectl --help`

## Names and Name Resolutions

- POSIX functions
  - `gethostname` uses `uname` syscall to query `utsname` and retruns
    `utsname::nodename`
  - `sethostname` uses `sethostname` syscall to set `utsname::nodename`
  - `getnameinfo`, which obsoletes `gethostbyaddr`, resolves addr to name
  - `getaddrinfo`, which obsoletes `gethostbyname`, resolves name to addr
    - the behavior is configured by `/etc/nsswitch.conf`
- GLIBC Resolver
  - `man resolver` and `man resolv.conf`
  - DNS name resolution
  - `getaddrinfo` can be configured to use the resolver
- POSIX tools
  - `hostname` calls `gethostname` to return the hostname
    - kernel defaults to `CONFIG_DEFAULT_HOSTNAME`, which is usually `(none)`
    - systemd calls `sethostname` with the first of
      - `systemd.hostname=` from kernel cmdline
      - `/etc/hostname`
        - `man 5 hostname` recommends a single label without any dot
        - some apps might break though
      - DHCP lease
      - hard-coded `localhost`
  - `domainname`
    - this is NIS domain name and is irrelevant nowadays
  - `dnsdomainname` and FQDN
    - FQDN is `addrinfo::ai_canonname` queried from `getaddrinfo`
    - the part after the first dot is the DNS domain name
- `getaddrinfo` and `nsswitch.conf`
  - systemd recommendations for name resolution
    - `hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns`
  - `mymachines` resolves with `systemd-machined` for vms/containers
  - `resolve` resolves with `systemd-resolved` but may not be available
    - `!UNAVAIL=return` means, if the return code is not `UNAVAIL`, return
      immediately
  - `files` resolves with `/etc/hosts`
    - systemd recommends a short nickname for `/etc/hostname`, which is not
      used here
    - on the othe hand, both the canonical name as well as the short nickname
      (alias) should be listed in `/etc/hosts`
    - this way, `getaddrinfo` can resolve the short nickname to the canonical
      name, which is the FQDN of the local machine
  - `myhostname` resolves local names
    - it resolves the current hostname to `127.0.0.2`
    - it resolves `localhost`, `localhost.localdomain`, `.localdomain`, and
      `.localhost.localdomain` to `127.0.0.1`
    - it resolves `_gateway` to the current default gateway
    - it resolves `_outbound` to the current ip addr
  - `dns` resolves with DNS configured by `/etc/resolv.conf`
