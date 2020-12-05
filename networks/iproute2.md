iproute2
========

## Static IP and Routing

- `ip addr add 192.168.0.1/24 dev eth0`
- `ip route add 192.168.0.0/24 dev eth0`
- `ip route add default via <gateway>`

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
