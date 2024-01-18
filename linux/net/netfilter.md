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
