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

## Manual Link Setup without systemd-networkd

- for wired, automatic or `ip link set eth0 up`
- for wireless with `iwd`
  - `systemctl enable --now iwd`
  - `iwctl`
    - `device list`
    - `station <dev> scan`
    - `station <dev> get-networks`
    - `station <dev> connect "<ssid>"`
      - password will be prompted
  - the link and password will be saved to `/var/lib/iwd`
- for wireless with `wpa_supplicant`
  - create `/etc/wpa_supplicant/wpa_supplicant-<iface>.conf`

      network={
        ssid="<ssid>"
        psk="<password>"
      }
  - `systemctl enable --now wpa_supplicant@<iface>`

## Manual Network Configuration without systemd-networkd

- for static ip,
  - `ip addr add 192.168.0.1/24 dev eth0`
  - `ip route add 192.168.0.0/24 dev eth0`
  - `ip route add default via <gateway>`
- for dynamic ip,
  - `dhcpcd --debug --nobackground eth0`
- `iwd` also has a built-in DHCP client, if desirable
  - create `/etc/iwd/main.conf`

      [General]
      EnableNetworkConfiguration=true

## Automatic Network Configuration with systemd-networkd

- `systemctl enable --now systemd-networkd`
- a `.network` file can be added to `/etc/systemd/network` to manage a link
  - it normally contains

      [Match]
      Name=eth0
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
- there are also `DHCPServer` and `IPMasquerade` that can automate NAT setup?
  - for manual setup, see <dnsmasq.md> and <nftables.md>

## systemd.netdev

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
- see <tuntap.md> for manual setup
- a network can be further configured using a `.network` file
  - Does a bridge need an IP? It depends.
    - yes when the bridge is also used as the gateway of the networks in
      containers

## systemd-resolved

- systemd-resolved can manage `/etc/resolv.conf` or just be a client of
  `/etc/resolv.conf`
- when systemd-networkd receives DNS servers from the DHCP server, it tells
  `systemd-resolved`
  - in manual setup, `dhcpcd` invokes `resolvconf`
- `resolvectl status`

## ifup / ifdown

- ifup / ifdown uses `/etc/network/interfaces`
- systemd-networkd replaces ifup / ifdown

## NetworkManager

- NetworkManager competes with systemd-networkd and is more suitable for
  laptops
- see <NetworkManager.md>
