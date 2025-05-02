TCP/UDP Ports
=============

## Protocols

- ssh: tcp 22
- dns: tcp/udp 53
- dhcp/bootp: udp 67 (server), udp 68 (client)
- http: tcp 80
- https: tcp 443
- dhcpv6: udp 547 (server), udp 546 (client)
- ipp: tcp 631
- upnp/ssdp: udp 1900
  - win also uses tcp 2869 or 5000
- mdns: udp 5353
- llmnr: tcp/udp 5355
- wireguard: udp 51820

## Applications

- airsane
  - http: tcp 8090
- avahi
  - mdns: udp 5353
  - wide-area dns-sd: udp random
- cups
  - ipp: tcp 631
- home assistant
  - upnp: udp 1900
  - mdns: udp 5353
  - http: tcp 8123
  - upnp: tcp 40000
  - random tcp/udp ports
- jellyfin
  - upnp: udp 1900
  - service discovery: udp 7359
  - http: tcp 8096
  - https: tcp 8920
- kavita
  - http: tcp 5000
- nginx
  - http: tcp 80
- omada
  - https: tcp 8043
  - http: tcp 8088
  - https: tcp 8843
- prometheus
  - http: tcp 9100 (node exporter)
  - http: tcp 9090 (prometheus)
  - http: tcp 3000 (grafana)
- systemd-journal-remote
  - http: tcp 19532
