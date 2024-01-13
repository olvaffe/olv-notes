Homelab
=======

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
  - it is a name resolver
    - it resolves locally first, and if that fails, it forwards the requests
      to the dns servers
  - it provides 3 interfaces
    - the native interface is d-bus
    - `nsswitch.conf` can be configured to use `systemd-resolved`
    - it listens on dns ports as well
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
- VLAN (802.11q)
  - an untagged/access port is a port where the incoming and outgoing frames
    are untagged
    - that is, the device connected to the port sends/receives untagged frames
    - the port is configured with a specific vlan id
      - for each incoming frame received from the connected device, the switch
        adds a tag, checks the destination, and forwards the frame to the
        right port
      - for each outgoing frame forwarded from another port, the switch checks
        the tag, strips the tag, and sends the frame
        - if the frame is untagged or has a wrong tag, the frame is dropped
          instead
  - a tagged/trunk port is a port where the incoming and outgoing frames
    are tagged (except for native vlan)
    - that is, the device connected to the port sends/receives tagged frames
      (e.g., another switch's tagged port) except for native vlan
    - the port is configured with multiple vlan ids
      - for each incoming frame received from the connected device, the switch
        checks the tag, checks the destination, and forwards the frame to the
        right port
      - for each outgoing frame forwarded from another port, the switch checks
        the tag, and sends the frame
    - the port is also configured with a native vlan id
      - it works the same way as an untagged port
  - no first-hand experience, but it feels like each port has a native vlan id
    and a list of allowed vlan ids
    - if a frame is not tagged, it is assumed to have the native vlan id

## HW

- Cabling
  - Cat6, 23 AWG, U/UTP, CMP (or at least CMR): 250ft
  - Cat6 RJ45 connector: x2
  - Cat6 punch down keystones: x2+4
  - Wall plates: x2
  - 8-port patch panel: x1
  - Cat6 patch cable: x4
