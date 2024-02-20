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

## Rules

- each rule consists of zero or more expressions followed by one or more statements
  - each expression tests whether a packet matches a specific payload field or
    packet/flow metadata
  - multiple expressions are linearly evaluated from left to right: if the
    first expression matches, then the next expression is evaluated and so on
  - if we reach the final expression, then the packet matches all of the
    expressions in the rule, and the rule's statements are executed
  - each statement takes an action, such as setting the netfilter mark,
    counting the packet, logging the packet, or rendering a verdict such as
    accepting or dropping the packet or jumping to another chain
  - as with expressions, multiple statements are linearly evaluated from left
    to right: a single rule can take multiple actions by using multiple
    statements
  - do note that a verdict statement by its nature ends the rule
- expressions evalute to typed values
  - they can be constants
  - the can be derived from packet metadata or payload
  - they can be combined from other expressions
- primary expressions
  - `meta` refers to metadata associated with a packet
  - `socket` refers to socket associated with a packet
  - `osf` refers to os fingerprinting associated with a packet
  - `fib` refers to fib (forwarding info base) associated with a packet
  - `rt` refers to routing data associated with a packet
  - `ipsec` refers to ipsec data associated with a packet
  - `numgen`, `jhash`, and `symhash` generate a number, usually used for
    load-balancing
- payload expressions refer to data from the packet payload
  - `ether` refers to ethernet header fields
  - `vlan` refers to vlan header fields
  - `arp` refers to arp header fields
  - `ip` refers to ipv4 header fields
  - `icmp` refers to icmp header fields
  - `igmp` refers to igmp header fields
  - `ip6` refers to ipv6 header fields
  - `icmpv6` refers to icmpv6 header fields
  - `tcp` refers to tcp header fields
  - `udp` refers to udp header fields
  - `udp-lite` refers to udp-lite header fields
  - `sctp` refers to sctp header fields
  - `dccp` refers to dccp header fields
  - `ah` refers to authentication header fields
  - `esp` refers to encrypted security payload header fields
  - `comp` refers to ipcomp header fields
  - `gre` refers to gre header fields
  - `geneve` refers to geneve header fields
  - `gretap` refers to encapsulated ethernet frame within the gre header
  - `vxlan` refers to vxlan header fields
  - `@base,offset,len` refers to raw bytes
  - a bunch of extension header expressions refer to variable-sized protocol
    headers
  - `ct` refers to conntrack metadata associated with a packet
- statements
  - `accept`, `drop`, `queue`, `continue`, `return`, `jump`, and `goto` alter
    the control flow
    - there are known as verdict statements
  - `<payload-expr> set <val>` alters packet content
  - `log` logs a packet
  - `reject` drops the packet and sends back an error packet
  - `counter` updates the hit count
  - `ct` sets conntrack marks/labels
  - `notrack` disables conntrack for a packet
  - `meta` sets a metadata
  - `limit` matches a packet until the limit is reached
    - if `over` is specified, it matches a packet after the limit is exceeded
  - `snat`, `dnat`, `masquerade`, and `redirect` alter saddr/daddr
  - `tproxy` redirects a packet to another local socket
  - `synproxy`
  - `flow`
  - `queue` passes a packet to userspace over `nfnetlink_queue`
  - `dup` duplicates a packet and sends the copy to a different dst
  - `fwd` redirects a packet to another dst
  - `add` and `update` adds a packet to a set
    - the difference is `update` refreshes the timeout
  - `map` performs table lookup
  - `vmap` is like a map, except that the values are verdict statements
  - `xt` executes a legacy xtables statement

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
