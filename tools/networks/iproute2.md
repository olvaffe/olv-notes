iproute2
========

## Repo

- <https://git.kernel.org/pub/scm/network/iproute2/iproute2.git>
  - use `AF_NETLINK`/`NETLINK_ROUTE` to communicate with the kernel
- replaces `net-tools`
  - <https://net-tools.sourceforge.io/>
  - use ioctl to communicate with the kernel
  - `arp` -> `ip neigh`
  - `ifconfig` -> `ip addr`
  - `ipmaddr` -> `ip maddr`
  - `iptunnel` -> `ip tunnel`
  - `route` -> `ip route`
  - `netstat` -> `ss`
- replaces `bridge-utils`
  - <https://git.kernel.org/pub/scm/network/bridge/bridge-utils.git>
  - use ioctl to communicate with the kernel
- does not replace `iputils`
  - <https://github.com/iputils/iputils>
  - `ping`, `tracepath`

## OSI

- L7 - Application layer
  - e.g., HTTP, SSH
- L6 - Presentation layer
- L5 - Session layer
- L4 - Transport layer
  - e.g., TCP, UDP
- L3 - Network layer
  - e.g., IPv4, IPv6
- L2 - Data link layer
  - e.g., Ethernet MAC
- L1 - Physical layer
  - e.g., Ethernet PHY interfaces between analog and digital
    - it connects to MAC over MII

## Link Config

- ethernet
  - `ip link set eth0 up`
- wifi
  - see `iwd` or `wpa_supplicant` in <wifi.md>

## IP and Routing

- for static ip,
  - `ip addr add 192.168.0.1/24 dev eth0`
  - `ip route add 192.168.0.0/24 dev eth0`
  - `ip route add default via <gateway>`
- DHCPv4
  - `systemd-networkd` supports DHCPv4
  - `iwd` also has a built-in DHCPv4 client, if desirable
    - create `/etc/iwd/main.conf`

        [General]
        EnableNetworkConfiguration=true
  - there is also `dhcpcd --debug --nobackground eth0`
- IPv6 RA and DHCPv6
  - `systemd-networkd` supports both and enables RA by default

## Socket Statistics

- to show all listenting tcp sockets and the processes,
  - `sudo ss -l -t -p`

## TAP

- to add a TAP device, tap0,
  - `sudo ip tuntap add tap0 mode tap user $(whoami)`
  - `sudo ip link set tap0 up`
- tap0 is added to kernel networking, but the actual sending/receiving of
  ethernet frames must be done by the userspace (to emulate a HW)
  - open `/dev/net/tun`
  - `ioctl(TUNSETIFF)` to control tap0
- experiment
  - assign an address to tap0
    - `sudo ip addr add 192.168.0.1 dev tap0`
  - set up routing
    - `sudo ip route add 192.168.0.0/24 via 192.168.0.1 dev tap0`
  - inside an vm, assign 192.168.0.2 to the emulated device
  - I guess, when the host pings 192.168.0.2, the host kernel sends a ethernet
    frame via tap0 (an ARP packet first, I guess).  The hypervisor receives
    the frame and forwards the frame to the emulated device in the VM.  The
    guest kernel receives the frame via the emulated device, handles the
    frame, and sends back another frame.  This response frame is received by
    the hypervisor via the emulated device and forwarded to the host kernel
    via tap0.  ping receives the response.

## VETH

- veth comes in pairs
  - the two devices in each pair are connected together
- to add a veth pair,
  - `ip link add <p1-name> type veth peer name <p2-name>`
- to move one of the device to another namespace,
  - `ip link set <p2-name> netns <p2-ns>`
  - this allows communications between network namespaces
- comparing to tap, no userspace handling of frames is needed
  - tap is for VMs, with hypervisor emulated HWs
  - veth is for containers

## Bridge

- in osi model,
  - hubs are L1 (physical layer) devices
  - switches are L2 (data link layer) devices
    - they understand ethernet frames
    - they inspect the mac addrresses and forward frames to the right ports
  - routers are L3 (network layer) devices
    - they understand ip packets
- bridges are switches
  - the term bridge is mostly only used in ieee specs
  - some may refer to a 2-port switch as a bridge
  - the bottom line is they are the same thing
- in this physical network, `host <-> switch <-> router <-> internet`,
  - when the host wants to send a ip packet to the internet, it adds the
    ethernet frame header to the ip packet and sends it out
    - the mac src is the host
    - the mac dst is the router
  - the switch checks the mac dst and forwards the frame to the router
  - the router checks the ip dst and route it to the final dst
- linux bridge
  - `ip link add br0 type bridge`
  - `ip link set eth0 master br0`
  - bridge mac addr
    - if manually assigned, it will never change
    - if generated, it will change as ports are added/removed
      - it uses the lowest mac addr of all ports, or zero if no port
      - see `br_stp_recalculate_bridge_id`
- I guesss,
  - just like a physical switch is not a netdev, a virtual bridge is also not
    an netdev
  - `br0` is more like a virtual nic connected to the bridge, than the bridge
    itself
  - when another netdev is added to the bridge, the netdev becomes a port
    - the netdev is still available, but is only used to configure the port
    - the netdev is no longer involved in routing
- <https://hicu.be/bridge-vs-macvlan>
