Linux net core
==============

## Initialization

- `subsys_initcall(net_dev_init)`
  - `dev_proc_init`
    - `register_pernet_subsys(&dev_proc_ops)`
      - when a new net namespace is created, `dev_proc_net_init` is called to
        populate `/proc/net`
        - `/proc/net/dev` shows all devices in `net`, the net namespace
        - `/proc/net/ptype` shows all packet handlers
          - an ethernet frame consists of
            - MAC dst
            - MAC src
            - optional vlan tag
            - `EtherType`
              - this specifies the payload data type
            - payload
            - crc
          - different packet handlers are registered with `dev_add_pack` for
            different `EtherType`
        - `/proc/net/wireless` shows all wireless info
    - `register_pernet_subsys(&dev_mc_net_ops)`
      - `/proc/net/dev_mcast` shows the multicast info
  - `netdev_kobject_init`
    - `kobj_ns_type_register(&net_ns_type_operations)`
    - `class_register(&net_class)` for `/sys/class/net`
  - `register_pernet_subsys(&netdev_net_ops)`
    - `netdev_init` initialises `struct net`, the new net namespace
  - `register_pernet_device(&loopback_net_ops)`
    - `loopback_net_init` creates `lo` for each new net namespace
      - `alloc_netdev` allocates a `struct net_device`
      - `register_netdev` registers the `struct net_device`
  - `register_pernet_device(&default_device_ops)`
    - `default_device_exit_batch` removes all devices before a net namespace
      is destroyed
  - `open_softirq(NET_TX_SOFTIRQ, net_tx_action)` opens a softirq for the
    bottom-half of tx irq
    - the kernel allocates a ring buffer and writes outgoing data to it
    - after the hw reads and sends the outgoing data, it generates an
      interrupt
  - `open_softirq(NET_RX_SOFTIRQ, net_rx_action)` opens a softirq for the
    bottom-half of rx irq
    - the kernel allocates a ring buffer
    - after the hw receives and writes the incoming data to the ring buffer,
      it generates an interrupt

## Ethernet Frame RX Flow

- when the hw receives an ethernet frame, it generates an interrupt
- the driver handles the interupt and calls `napi_schedule`
- `____napi_schedule` adds `napi_struct` to `sd->poll_list` and raises
  `NET_RX_SOFTIRQ`
- `net_rx_action` calls `napi_poll` on all entries of `sd->poll_list`
- `__napi_poll` calls the driver's `poll` callback
  - the driver has initialized the napi with `netif_napi_add` and provided the
    callback
- in the poll function, the driver
  - asks the hw to copy the frame to cpu memory
  - allocates and initializes a `sk_buffer` (metadata for the frame)
  - calls `napi_gro_receive`, generic receive offloading
    - some drivers call `netif_receive_skb` instead
  - `napi_skb_finish` calls `netif_receive_skb_list_internal`
  - `__netif_receive_skb_list` calls `__netif_receive_skb_list_core` which
    calls `__netif_receive_skb_core` on each skb
  - `deliver_skb` calls the packet handler, `packet_type::func`
    - it dispatches to different handlers depending on the `EtherType` of the
      frame
    - e.g., `ip_rcv` for ipv4 and `ipv6_rcv` for ipv6
- `ip_rcv` handles received ipv4 packets
  - `ip_rcv_core` parses the ip header
  - nftable `NFPROTO_IPV4`/`NF_INET_PRE_ROUTING` hook is run
  - `ip_rcv_finish` calls `ip_route_input_noref` to route the packet
    - it sets the `input` callback to `ip_local_deliver`, `ip_forward`, or
      others
  - `dst_input` calls the `input` callback
- `ip_local_deliver`
  - nftable `NFPROTO_IPV4`/`NF_INET_LOCAL_IN` hook is run
  - `ip_protocol_deliver_rcu` calls `inet_protos[protocol]->handler`
    - the array is modified by `inet_add_protocol`
    - protocols are `IPPROTO_TCP`, `IPPROTO_UDP`, `IPPROTO_ICMP`, etc.
- `tcp_v4_rcv` handles received tcp packet
  - `__inet_lookup_skb` finds the `struct sock` for the packet src/dst
  - `tcp_v4_fill_cb` parses the packet header
  - `tcp_add_backlog` adds the skb to the sk backlog
- later, when the socket is read, `tcp_recvmsg` copies the packet to the
  userspace buffer
  - `__sk_flush_backlog` flushes the backlog
    - `sk_backlog_rcv` points to `tcp_v4_do_rcv`
    - if the connection is already established, `tcp_rcv_established` calls
      `tcp_queue_rcv` to queue the packet for userspace and calls `tcp_ack` to
      ack the packet
  - `skb_copy_datagram_msg` copies the packet from skb to msg
