Homelab
=======

## Home Network

- modem
  - 1Gb uplink
  - 1Gb ethernet port
- router
  - pfsense netgate advertises L3 forwarding, firewall, and vpn numbers
  - fanless home lab
- switch
  - unlike hub, a switch inspects the mac addrress and forwards packets to the
    right ports
  - power consumption
    - 1Gb x8, max 3W
      - if poe x4, add 3W
      - if poe x8, add 6W
      - if smart, add 3W
      - if managed (vlan, 802.1x, etc.), no difference
    - 2.5Gb x8, max 7.5W
    - 10Gb x8, max 30W, active cooling
- ap
  - x2, poe+
- Cabling
  - Cat6a, 23 AWG, U/UTP, CMP: 250ft
  - Cat6a RJ45 connector: x2
  - Cat6a punch down keystones: x2+4
  - Wall plates: x2
  - 8-port patch panel: x1
  - Cat6 patch cable: x4
  - 12.5ft telescoping extension ladder
  - or, moca adapters for coax

## Distros

- type-1 hypervisor
  - proxmox
    - debian-based
    - x86-only (there is unofficial arm port)
    - upgrade can be done via apt or reinstall
      - apt preverses configs
      - but backup is always recommended
- router
  - openwrt
    - linux-based
    - x86 and arm
    - wired and wireless
    - many packages
    - upgrade wipes the rootfs, flashes the new firmware, and reboots
      - the firmware must be downloaded manually
      - all configs/packages are lost!
      - backup and restore can be easy if customizations are limited to UCI
        (`/etc/config`)
  - pfesnse
    - freebsd-based
    - x86 and arm
    - wired only
    - only routing-related packages
    - upgrade can be triggered from webgui or from console
      - it downloads the update, applies it, and reboots
      - it tries to preserve configs
        - but backup is always recommended
- general purpose
  - armbian
  - debian
    - `unattended-upgrades`

## Networking

- types of devices
  - trusted clients (laptops, phones, etc.)
  - untrusted clients (most iot)
  - servers (ssh, cast, etc.)
  - network management (dhcp, dns, omada, etc.)
- WAN
  - ISP is Xfinity
  - need to reboot the modem to get a new DHCP lease
  - modem provides a private network, `192.168.100.0/24`, briefly during boot
  - xfinity provides
    - an ipv4 address in `73.92.126.0/23`
    - two ipv6 addresses (from RA and from DHCPv6 respectively?)
    - a `/60` delegated prefix (it does not include the two ipv6 addresses)
  - `20-wan.network`
    - `DHCP=ipv4` suffices
    - ipv6 ra is enabled by default and will trigger dhcpv6
- LAN
  - `20-lan.network`
    - `Address=192.168.0.254/24`
    - `DHCPServer=yes` to start dhcpv4
    - `IPForward=yes` to forward both ipv4 and ipv6
    - `IPMasquerade=ipv4` for ipv4 nat
    - `DHCPPrefixDelegation=yes` to be assigned an ipv6 subnet
    - `IPv6SendRA=yes` for ipv6 ra
    - `IPv6AcceptRA=no`
- `systemd-resolved`
  - `20-lan.network`
    - `[Network]`
    - `DNS=75.75.75.75 75.75.76.76 2001:558:feed::1 2001:558:feed::2` from
      xfinity
    - `[DHCPServer]`
    - `DNS=_server_address` to advertise itself
    - `[IPv6SendRA]`
    - `EmitDNS=no` to not advertise wan dns server
  - `resolved.conf`
    - `DNSStubListenerExtra=192.168.0.254` to listen on the interface
  - `/etc/hosts` for local host names
