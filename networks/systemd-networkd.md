systemd-networkd
================

## overview

- systemd-networkd manages links and networks
- a link is managed by systemd-networkd only when there is a `.network` file
  under `/etc/systemd/network` (and other places)
  - the `.network` has lines such as these to match a link
    - `[Match]`
    - `Name=eth0`
- after a link is ready, a network is configured according to the `.network` file
  - for wired, the link is ready when it is connected
  - for wireless, the link is ready when `wpa_supplicant` or `iwd` sets it up
- `networkctl` lists all links and their states

## ifup / ifdown

- ifup / ifdown uses `/etc/network/interfaces`
- systemd-networkd replaces ifup / ifdown

## NetworkManager

- NetworkManager competes with systemd-networkd and is more suitable for
  laptops where the network environment changes dynamically
- see <NetworkManager.md>

## `systemd.link`

- a `.link` file can be added to `/etc/systemd/network` to manage a link
  - this is used by `systemd-udevd` rather than `systemd-networkd`
- it can match by mac address and configure the iface name, mtu size, etc.

## `systemd.network`

- `systemctl enable --now systemd-networkd`
- a `.network` file can be added to `/etc/systemd/network` to manage a network
  - it normally contains

      [Match]
      Name=eth0
      # or match by type: ether, wlan, etc.
      Type=ether
      [Network]
      # static
      Address=192.168.0.1/24 
      Gateway=192.168.0.254
      DNS=8.8.8.8
      # dynamic
      DHCP=yes
      # add to brdige
      Bridge=br0
- a network is configured only after the matching link is ready
  - automatic for wired
  - manual for wireless
- there are also `DHCPServer` and `IPMasquerade` that can automate NAT setup
  - for manual setup, see <dnsmasq.md> and <nftables.md>
- `[DHCPv4]` configures the DHCPv4 client, if enabled with `DHCP=`
  - `SendHostname=` and `Hostname=` set the hostname sent to the DHCPv4 server
    - on google wifi, which provides both DHCPv4 and DNS, the hostname is used
      to create an A record for `<hostname>.lan`
  - `UseDomains=` controls whether domain names received are used

## `systemd.netdev`

- a `.netdev` file can be added to `/etc/systemd/network` to create a virtual
  link
  - it normally contains

      [NetDev]
      # bridge
      Name=br0
      Kind=bridge
      # tap
      Name=tap0
      Kind=tap
      # veth
      Name=veth0
      Kind=veth
      [Peer]
      Name=veth0peer
- see <iproute2.md> for manual setup
- a network can be further configured using a `.network` file
- Does a bridge need an IP? It depends.
  - a bridge is a switch
    - frames received on one port are forwarded and sent from another port
    - ifaces added to `br0` become ports on the switch
  - `br0` is both the switch itself as well as an iface connecting to the
    switch
    - `br0` refers to the switch itself when we do switch management
    - `br0` refers to an iface connecting to the switch when we assign ip,
      routing, etc.

## D-Bus

- `busctl tree org.freedesktop.network1` to see all objects
  - `/org/freedesktop/network1/link/*` are links (interfaces)
  - `/org/freedesktop/network1/network/*` are networks
    - object names are derived from filenames (e.g., `20-wan.network` becomes
      `_320_2dwan`)

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
- special-use domain names
  - <https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml>
  - `in-addr.arpa.` for ipv4 reverse-mapping
  - `ip6.arpa.` for ipv6 reverse-mapping
  - `test.` is for testing
  - `local.` is for mdns
  - `invalid.` is always invalid
  - private network
    - `home.arpa.` is for residential private network
    - `localhost.` is for local private network
    - RFC6762 has some non-standard tlds for private networks
      - `intranet.`
      - `internal.`
      - `private.`
      - `corp.`
      - `home.`
      - `lan.`
    - redhat uses `localdomain.`
    - openwrt uses `lan.`

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
