nftables
========

## Basics

- listing
  - `nft list ruleset` lists current ruleset
  - `nft list tables` lists current tables
  - `nft list table <family> <name>` lists chains in a table
  - `nft list chain <family> <table-name> <chain-name>` lists rules in a chain
- save and restore
  - `echo "flush ruleset" > saved`
  - `nft -s list ruleset >> saved`
  - `nft -f saved`
- tables
  - to create a table,
    - `nft add table <family> <name>`
    - family is ip, ip6, inet, arp, bridge, etc.
  - to delete a table,
    - `nft delete table <family> <name>`
  - to flush a table,
    - `nft flush table <family> <name>`
- chains
  - to create a base chain in a table,
    - `nft add chain <family> <table_name> <chain_name> '...'`
    - a base chain is an entrypoint for a packet
  - to create a regular chain in a table,
    - `nft add chain <family> <table_name> <chain_name>`
    - a regular chain is like a helper function

## Address Families

- concept
  - kernel processes network packets
  - there are different types of packets
    - nft address families match different types of packets
  - there are different stages in the processing paths of different types of
    packets
    - nft hooks are called at the stages
- `ip`, `ip6`, and `inet` address families
  - `prerouting` hook
    - all packets entering the system are processed by this hook
  - `input` hook
    - packets delivered to the local system are processed by this hook
  - `forward` hook
    - packets forwarded to a different host are processed by this hook
  - `output` hook
    - packets sent by local processes are processed by this hook
  - `postrouting` hook
    - all packets leaving the system are processed by this hook
  - `ingress` hook
    - all packets entering the system are processed by this hook
    - called before `prerouting` hook
- `arp` address family
  - `input` and `output` hooks
- `bridge` address family
  - same hooks as `ip`/`ip6`/`inet`
- `netdev` address family
  - `ingress` and `egress` hooks

## SNAT

- `sysctl net.ipv4.ip_forward=1`
- iptables
  - `iptables -t nat -F` to flush all rules
  - `iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -j MASQUERADE`
    - `192.168.0.0/24` is the private network
- nftables

    flush ruleset
    table ip nat {
        chain POSTROUTING {
            type nat hook postrouting priority srcnat; policy accept;
            oifname "wlp2s0" masquerade
        }
    }

## `iptables`

- the kernel provides two interfaces
  - the older xtables which consists of iptables, ip6tables, arptables, etc.
  - the newer nftables which is just nftables
- the userspace provides two sets of tools
  - `iptables-legacy` is a symlink to `xtables-legacy-multi`, which uses the
    older xtables interface
  - `iptables-nft` is a symlink to `xtables-nft-multi`, which uses the newer
    nftables interface
  - `iptables` is a symlink to either `iptables-legacy` or `iptables-nft`
  - but `nft` should be used directly nowadays
- `iptables` can manipulate 5 tables, each has their own chains
  - `filter` is the default when `-t` is not specified
    - `INPUT`
    - `FORWARD`
    - `OUTPUT`
  - `nat`
    - `PREROUTING`
    - `INPUT`
    - `OUTPUT`
    - `POSTROUTING`
  - `mangle`
    - `PREROUTING`
    - `INPUT`
    - `FORWARD`
    - `OUTPUT`
    - `POSTROUTING`
  - `raw`
    - `PREROUTING`
    - `OUTPUT`
  - `security`
    - `INPUT`
    - `FORWARD`
    - `OUTPUT`
- `iptables -t <table> -L` list all rules in the table
- for a simple firewall, we only need to manipulate `filter` table
  - `INPUT`
    - `iptables -P INPUT DROP` sets the policy drop, meaning an ingress packet
      is dropped unless it is explicitly accepted by a rule
    - `iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`
      accepts any ingress packet that has state `RELATED` or `ESTABLISHED`
      - the other two states are `NEW` and `INVALID`
    - `iptables -A INPUT -i lo -j ACCEPT` accepts any ingress packet on `lo`
    - `iptables -A INPUT -p icmp -j ACCEPT` accepts ingress packets that are
      ping
    - `iptables -A INPUT -p udp --dport 5353 -j ACCEPT` accepts ingress
      packets to udp port 5353 (mdns)
    - `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` accepts ingress packets
      to tcp port 22 (ssh)
  - `FORWARD`
    - `iptables -P FORWARD DROP` sets the policy to drop, meaning an forwarded
      packet is dropped unless it is explicitly accepted
  - `OUTPUT`
    - `iptables -P OUTPUT ACCEPT` sets the policy to accept, meaning an egress
      packet is accepted unless it is explicitly dropped
