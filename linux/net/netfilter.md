Linux netfilter
===============

## Userspace

- <https://git.netfilter.org/nftables/>
  - `nft` is the only executable
  - dependencies
    - `libmnl` (`mnl_*`), a minimal netlink library
    - `libnftnl` (`nftnl_*`)
    - optional `libxtables` (`xtables_*`), for legacy xtables extension compat
  - the kernel header is mainly `nf_tables.h`
  - the socket is `AF_NETLINK`/`NETLINK_NETFILTER`
- <https://git.netfilter.org/iptables/>
  - there are two sets of tools, using the newer nftables and the legacy
    xtables as backends respectively
  - `{ip,ip6,arp,eb}tables-nft` are symlinks to `xtables-nft-multi`
    - `libmnl` and `libnftnl` are used to communicate with the kernel
  - `{ip,ip6}tables-legacy` are symlinks to `xtables-legacy-multi`
    - the kernel headers are mainly `{ip,ip6,x}_tables.h`
    - `libiptc` is used to communicate with the kernel
    - the socket is `AF_{INET,INET6}`/`IPPROTO_RAW`
    - `setsockopt` is used to update the rules
- <https://git.netfilter.org/arptables/>
  - `arptables-legacy` is the only executable
  - the kernel headers are mainly `{arp,x}_tables.h`
  - `libarptc` is used to communicate with the kernel
  - the socket is `AF_INET`/`IPPROTO_RAW`
- <https://git.netfilter.org/ebtables/>
  - `ebtables-legacy` is the only executable
  - the kernel header is mainly `netfilter_bridge.h`
  - `libebtc` is used to communicate with the kernel
  - the socket is `AF_INET`/`PF_INET`

## Hooks

- there are these address families
  - `NFPROTO_INET`
  - `NFPROTO_IPV4`
  - `NFPROTO_ARP`
  - `NFPROTO_NETDEV`
  - `NFPROTO_BRIDGE`
  - `NFPROTO_IPV6`
- these are these hooks
  - inet/ipv4/ipv6 have `enum nf_inet_hooks`
    - `NF_INET_PRE_ROUTING`
    - `NF_INET_LOCAL_IN`
    - `NF_INET_FORWARD`
    - `NF_INET_LOCAL_OUT`
    - `NF_INET_POST_ROUTING`
  - arp has
    - `NF_ARP_IN`
    - `NF_ARP_OUT`
    - `NF_ARP_FORWARD`
  - netdev has `enum nf_dev_hooks`
    - `NF_NETDEV_INGRESS`
    - `NF_NETDEV_EGRESS`
  - bridge has
    - `NF_BR_PRE_ROUTING`
    - `NF_BR_LOCAL_IN`
    - `NF_BR_FORWARD`
    - `NF_BR_LOCAL_OUT`
    - `NF_BR_POST_ROUTING`
- `nf_tables_module_init` calls `nft_chain_filter_init` to register various
  chain types
- `nf_register_net_hook` adds a hook to a netdev
  - `nf_hook_entry_head` returns different lists for different hooks for the
    netdev
- <https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks>
  - upon `netif_receive_skb`, `nf_hook_ingress` invokes `NF_NETDEV_INGRESS`
  - if bridge,
    - upon `br_handle_frame`, `nf_hook_bridge_pre` invokes `NF_BR_PRE_ROUTING`
    - upon `br_handle_frame_finish`,
      - if input, `br_pass_frame_up` invokes `NF_BR_LOCAL_IN`
        - upon `br_dev_xmit`, `br_forward` invokes `NF_BR_LOCAL_OUT`
      - if forward, `br_forward` invokes `NF_BR_FORWARD`
    - `br_forward_finish` invokes `NF_BR_POST_ROUTING`
  - if arp,
    - upon `arp_rcv`, it invokes `NF_ARP_IN`
    - upon `arp_send`, `arp_xmit` invokes `NF_ARP_OUT`
  - if inet/ipv4/ipv6,
    - upon `ip_rcv` or `ip_list_rcv`, it invokes `NF_INET_PRE_ROUTING`
    - upon `ip_rcv_finish_core`,
      - if input, `ip_local_deliver` invokes `NF_INET_LOCAL_IN`
        - upon `ip_local_out`, it invokes `NF_INET_LOCAL_OUT`
      - if forward, `ip_forward` invokes `NF_INET_FORWARD`
    - upon `ip_output`, `ip_output` invokes `NF_INET_POST_ROUTING`
  - upon `dev_queue_xmit`, `nf_hook_egress` invokes `NF_NETDEV_EGRESS`
