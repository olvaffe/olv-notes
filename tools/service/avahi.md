Avahi
=====

## Zeroconf

- <http://www.zeroconf.org/>
- Address Autoconfiguration
  - ipv4 uses `169.254.0.0/16` for link-local addressing
    - the host selects a random address in the range
    - the host sends an arp probe to see if the address is in use
    - if yes, the host selects another random address
  - ipv6 uses `fe80::/10` for link-local addressing
    - the host uses SLAAC (stateless address autoconfiguration) to select the
      address
  - ipv6 requires multiple addresses per interface and requires a link-local
    address to be assigned to each interface
    - other protocols such as NDP and DHCPv6 depend on the link-local address
  - ipv4 usually does not use the link-local address
    - ipv4 does not require multiple addresses per interface.  Hosts usually
      only assign link-local addresses as a last resort when a DHCP server is
      not available
    - an ipv4 link-local address is not useful unless the host also implements
      mdns and dns-sd
- Name Resolution
  - mDNS, multicast DNS, supports ipv4 and ipv6
    - it comes from apple and is a part of apple's Bonjour/Rendezvous service
    - microsoft has NetBIOS (ipv4 only) and LLMNR (Link-local Multicast Name
      Resolution), but is moving toward mdns now
  - when a host needs to resolve a name ending in `.local`,
    - it sends a multicast udp packet to
      - ipv4 address `224.0.0.251` or ipv6 address `ff02::fb`
      - port `5353`
    - the payload of the packet is based on the dns packet format
    - another host who owns the name identifies itself by multicasting another
      message with its ip
    - all hosts in the subnet updates their mdns caches
  - basically, it is dns without a centralized dns server but distributed
- Service Discovery
  - DNS-SD works over dns and mdns
  - it uses PTR/SRV/TXT records
    - `_ipp._tcp.local PTR my-printer._ipp._tcp.local`
      - this is for IPP service enumeration and is called "browsing" in the
        spec
      - there can be multiple such PTR records for each printer
      - clients request for `_ipp._tcp.local` and get a list of PTR records for
        printers on `.local` domain
    - `my-printer._ipp._tcp.local SRV 0 0 631 my-printer.local`
      - this is for service info
      - it says the ipp printer has priority 0, weight 0, port 631, and name
        `my-printer.local`
    - `my-printer._ipp._tcp.local TXT ...`
      - TXT is human-readable string, but is repurposed for machine-readable
        key/val pairs in dns-sd
      - <https://developer.apple.com/bonjour/printing-specification/bonjourprinting-1.2.1.pdf>
        - `note` is a utf8 string indicating the location of the printer
        - `pdl` is a list of common-seperated mime types supported by the
          printer
        - `rp` specifies the queue name.  Print jobs will be sent to
          `ipp://<hostname>:<port>/<rp>`
  - there are also special PTR records
    - a dns query for PTR recrods with the name `_services._dns-sd._udp.local`
      yields all services
  - `avahi-browse -atr` lists all services


## UPnP

- <http://www.zeroconf.org/ZeroconfAndUPnP.html>
  - addressing: both zeroconf and upnp use link-local addressing
  - naming: zeroconf uses mdns and upnp lacks the support
  - discovery: zeroconf uses dns-sd and upnp uses (abandoned) ssdp
  - application: zeroconf does not define application protocols and upnp
    focuses on application protocols
- DLNA
  - established in 2003 and disresolved in 2017
  - a server stores and streams contents
    - e.g., a nas server
  - a player browses and plays contents from servers
    - e.g., a game console
  - a controller browses contents from servers and controls players to play
    - e.g., a smartphone app
  - a printer prints contents from servers
    - e.g., print a photo from a camera to a printer
- Plex Media Server
  - mainly uses tcp port 32400
  - supports ssdp on udp port 1900
  - supports dns-sd on udp port 5353
