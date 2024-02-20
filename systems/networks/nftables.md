nftables
========

## Basics

- ruleset
  - `nft list ruleset` lists the ruleset
  - `nft flush ruleset` flushes the ruleset
  - save and restore
    - `echo "flush ruleset" > saved`
    - `nft -s list ruleset >> saved`
    - `nft -f saved`
  - `nft list tables` lists table names in the ruleset
- tables
  - `nft add table <family> <name>` creates a table
    - `family` is `ip`, `ip6`, `inet` (both `ip` and `ip6`), `arp`, `bridge`,
      etc.
  - `nft delete table <family> <name>` deletes a table
  - `nft list table <family> <table-name>` lists all chains and sets in a table
  - `nft flush table <family> <name>` flushes all chains and sets in a table
- chains in a table
  - `nft add chain <family> <table-name> <chain-name>` creates a chain
    - this form creates a regular chain
      - a regular chain is not attached to any netfilter hook and will be used
        like a helper function
    - adding `{ type <type> hook <hook> priority <prio>; policy <policy>; }`
      creates a base chian
      - a base chain is attached to `<hook>` and will be used like a `main`
        function
      - `type` is `filter`, `route`, `nat`, etc.
      - `hook` is `ingress`, `prerouting`, `input`, `forward`, `output`,
        `postrouting`, etc.
      - `prio` is an integer; a lower number has a higher priority
        higher priority
        - it can also be specified as `keyword +/- offset`
        - valid keywords and their meanings differ depending on `family`
        - for `inet`,
          - `raw` is -300
          - `mangle` is -150
          - `dstnat` is -100
          - `filter` is 0
          - `security` is 50
          - `srcnat` is 100
      - `policy` is `accept` (default) or `drop`
  - `nft delete chain <family> <table-name> <chain-name>` deletes a chain
  - `nft list chain <family> <table-name> <chain-name>` lists all rules in a
    chain
  - `nft flush chain <family> <table-name> <chain-name>` flushes (deletes) all
    rules in a chain
- rules in a chain
  - `nft add rule <family> <table-name> <chain-name> ...` adds a rule
  - `nft delete rule <family> <table-name> <chain-name> handle <handle>`
    delete a rule
    - `nft -a list chain <family> <table-name> <chain-name>` to see the rule
      handles
- sets in a table
  - `nft add set <family> <table-name> <set-name> { ... }` creates a set
    - `type` is `ipv4_addr`, `ipv6_addr`, `inet_proto`, etc.
    - `timeout` is the timeout before an element is deleted automatically
    - `flags` is
      - `constant` means the set is immutable
      - `interval` means elements can be intervals
      - `timeout` means the set has a timeout
      - `dynamic` means rules can add/remove elements
  - `nft delete set <family> <table-name> <set-name>` deletes a set
  - `nft list set <family> <table-name> <set-name>` lists all elements in a
    set
  - `nft flush set <family> <table-name> <set-name>` flushes (deletes) all
    elements a set
- elements in a set
  - `nft add element <family> <table-name> <set-name> ...` creates an element
  - `nft delete element <family> <table-name> <set-name> ...` deletes an element

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
