Linux socket
============

## Sockets

- socket(7)
- socket(2) returns an fd for an endpoint for communication
  - domain (or address family or protocol family) specifies the protocol
    family
    - `address_families(7)`
    - `AF_UNIX`
    - `AF_INET`
    - `AF_NETLINK`
    - `AF_BLUETOOTH`
  - type specifies the endpoint type
    - `SOCK_STREAM`
    - `SOCK_DGRAM`
    - `SOCK_RAW`
  - protocol specifies the protocol in the protocol family
    - usually 0
    - for `AF_NETLINK`
      - `NETLINK_ROUTE`
      - `NETLINK_SELINUX`
      - `NETLINK_ISCSI`
      - `NETLINK_NETFILTER`
      - `NETLINK_KOBJECT_UEVENT`
- bind(2) sets the address of the endpoint
  - for unix(7), the address is a file name
    - this creates a socket file (`S_IFSOCK`) in the fs
    - linux supports abstract name where no socket file is created
    - socketpair(2) creates two endpoints without any name
- connect(2) connects the endpoint to the address

