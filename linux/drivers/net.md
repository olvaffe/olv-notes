Linux net drivers
=================

## veth

- `ip link add ve-host type veth peer name ve-guest`
  - it is handled by `veth_link_ops`
  - `veth_setup` sets up a new `net_device`
  - `veth_newlink` initilizes the new link
    - it creates the peer first
      - `rtnl_create_link` create the peer, also with `veth_link_ops`
      - `register_netdevice` registers the peer
      - `rtnl_configure_link` configures the peer
    - `register_netdevice` registers the dev itself
    - the dev and the peer points to each other in their `peer` member
    - `veth_init_queues` is called on both the dev and the perr
- `veth_xmit` is called by `dev_hard_start_xmit` when there is an outgoing
  frame
  - `veth_forward_skb` forwards the skb to the peer
- `veth_poll` is called by `napi_poll` when there is an incoming frame
  - `napi_gro_receive` or `netif_receive_skb` is called to move the skb to the
    upper layer
