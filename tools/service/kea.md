Kea
===

## DHCP

- history
  - RARP, 1984
  - BOOTP, 1985
  - DHCP, 1993
  - DHCPv6, 2003, 2018
- DHCP DORA (discovery, offser, request, acknowledgement)
  - client broadcasts `DHCPDISCOVER` to `255.255.255.255`
    - udp client port 68, server port 67
  - server sends `DHCPOFFER`
  - client broadcasts `DHCPREQUEST`
  - server sends `DHCPACK`
- RFC 2132 lists DHCP Options
  - options can be included in all DHCP packets
    - server ignores unknown options
  - 1, subnet mask
  - 3, router
  - 6, dns
  - 12, host name
  - 15, domain name
  - 19-25, ip layer params per host
  - 26-33, ip layer params per interface
  - 34-36, link layer params per interface
  - 37-39, tcp params
  - 43, vendor specifics
  - 50, requested ip addr
  - 51, ip addr lease time
  - 53, dhcp message type
  - 54, server id
  - 55, param request list
  - 60, vendor class id
  - 66, tftp server name
  - 67, bootfile name
  - 255, end
  - many more

## Config

- `interfaces-config`
  - `service-sockets-require-all` makes sockets required
- `option-data`
  - `domain-name-servers` specifies dns servers
  - `domain-name` specifies domain name
