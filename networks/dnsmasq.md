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

