dnsmasq
=======

## overview

- dnsmasq has multiple subsystems
- DNS subsystem
  - local DNS based on `/etc/hosts` and others
  - forwarding of queries to upstream DNS servers
  - caching
- DHCP subsystem
  - static and dynamic leases are supported
  - full PXE server, with netboot menus and multi-arch support
  - built-in read-only TFTP

## Usage

- to disable DNS, `--port 0`
  - to explicitly specify a server, `--server 8.8.8.8`
- to enable DHCP, `--dhcp-range <range>`
- to advertise PXE boot options over DHCP, `--pxe-service <type,name>`
- to enable TFTP, `--enable-tftp`
- dnsmasq binds to the wildcard address by default
  - to bind to a certain interface, `--interface`
    - this adds `lo` automatically
  - to bind to a certain address, `--listen-address`
  - even when both options above present, dnsmasq still binds to the wildcard
    address.  It filters internally and responds to the binded
    interface/address only.  To change the behavior, specify
    `--bind-interfaces`

## PXE example

- example

    dnsmasq --no-daemon \
            --bind-interfaces \
            --listen-address 192.168.0.1
            --port 0 \
            --dhcp-range 192.168.0.10,192.168.0.20
            --log-dhcp \
            --pxe-service=0,"Raspberry Pi Boot" \
            --enable-tftp \
            --tftp-root <path>
- make sure `<path>` is accessible by nobody or specify `--user`

## Names

- `hostname`
  - kernel defaults hostname to `CONFIG_DEFAULT_HOSTNAME`, which is usually
    `(none)`
  - `systemd` uses first of
    - `systemd.hostname=` from kernel cmdline
    - `/etc/hostname`
      - `man 5 hostname` recommends a single label without any dot
      - some apps might break though
    - DHCP lease
    - hard-coded `localhost`
  - many apps calls `gethostbyname` to resolve the hostname
    - `/etc/nsswitch.conf` is normally configured to use `/etc/hosts` and then
      DNS
    - if hostname is a nickname, specify an alias in `/etc/hosts`
    - otherwise, dns lookup is configured by `/etc/resolv.conf`
- `domainname`
  - this is NIS domain name and is irrelevant nowaday
- `dnsdomainname` and FQDN
  - the canonical name returned by calling `getaddrinfo` for `gethostname` is
    the FQDN
  - the part after the first dot is the DNS domain name
  - `getaddrinfo` is configured by `/etc/nsswitch.conf`
    - in the end, the first name specified in `/etc/hosts` is FQDN
