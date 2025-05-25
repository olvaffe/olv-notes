systemd-networkd
================

## overview

- systemd-networkd manages links and networks
- a link is managed by systemd-networkd only when there is a `.network` file
  under `/etc/systemd/network` (and other places)
  - the `.network` has lines such as these to match a link
    - `[Match]`
    - `Type=ether`
    - `[Network]`
    - `DHCP=yes`
    - `MulticastDNS=yes`
    - `LLMNR=no`
    - `[DHCPv4]`
    - `UseDomains=yes`
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
      # dynamic ipv4
      DHCP=yes
      # dynamic ipv6
      IPv6AcceptRA=yes
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
