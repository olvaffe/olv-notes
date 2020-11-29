IPv6
====

## internal network

- called ULA (unique local address)
  - `fc00::/7`
  - similar to `192.168.0.0/16`
- let's use `fddd:59c2:214b::/48` for the router
  - `ip addr add fddd:59c2:214b::1/48 dev eth0`
- how do machines in the network get their addresses?
  - `radvd`

## 6to4

- suppose we have an IPv4 address, how do we connect to IPv6 network?
- suppose we have an IPv4 address, how do we listen to an IPv6 address?

## DHCPv6

- 
