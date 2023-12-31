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

## syscalls

- `socket(family, type, protocol)` creates a socket
  - `sock_create` creates the socket
    - `sock_alloc` allocates the socket which has an inode on `sockfs`
    - `family` is the index into the static `net_families` array
      - e.g., `AF_UNIX` maps to `unix_family_ops`
      - `sock_register` updates the array
  - `sock_map_fd` assocites the socket with a file
- `connect(sockfd, addr, addrlen)` makes a connection
  - this calls `proto_ops::connect`
- `bind(sockfd, addr, addrlen)` binds a name to a socket
  - this calls `proto_ops::bind`
- `listen(sockfd, backlog)` marks a socket accepting connections
  - this calls `proto_ops::listen`
- `accept(sockfd, addr, addrlen)` accepts a connection
  - this calls `sock_alloc` to allocate a new socket and calls
    `proto_ops::accept`
- `shutdown(sockfd, how)` shuts down a connection
  - this calls `proto_ops::shutdown`
- unix socket
  - `net_families[AF_UNIX]` points to `unix_family_ops`
  - `socket->ops` points to
    - `unix_stream_ops` if `SOCK_STREAM`
    - `unix_dgram_ops` if `SOCK_DGRAM`
    - `unix_seqpacket_ops` if `SOCK_SEQPACKET`
