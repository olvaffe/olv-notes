nftables
========

## Overview

- `man 8 nft` overview
- Options
  - `-f` to read cmds from file or stdin
  - `-c` to dryrun
    - otherwise, changes are applied atomically
  - `-a` to show object handles
  - `-s` to omit stateful info
  - `-j` to output in json
- Input File Formats
  - cmds are parsed line-wise
  - a cmd ends on newline or semicolon
  - a hash begins a comment until newline
  - `include "filename"` includes another file
  - `define var = expr` defines a variable, referred to by `$var`
- Address Families
  - `ip` is ipv4
    - `prerouting`
    - `input`
    - `forward`
    - `output`
    - `postrouting`
    - `ingress`
  - `ip6` is ipv6
    - same hooks as ipv4
  - `inet` is ipv4/ipv6
    - same hooks as ipv4
  - `arp` is ipv4 arp
    - `input`
    - `output`
  - `bridge` is bridge
    - same hooks as ipv4
  - `netdev` is netdev
    - `ingress`
    - `egress`
- Ruleset
  - `list ruleset` shows the ruleset
  - `flush ruleset` clears the ruleset
- Tables
  - `{add | create} table [family] table-name [{...}]` creates a table
    - `create` differs from `add` in that it fails if the table exists
    - it defaults to `add` if neither is specified
    - `family` defaults to `ip`
  - `{delete | destroy | list | flush} table [family] table-name`
    - `delete` differs from `destroy` in that it fails if the table does not
      exist
- Chains
  - `{add | create} chain [family] table-name chain-name [{...}]` creates a
    chain
    - it can also nest under `table {...}`
      - `add`, `family`, and `table-name` are always implied
  - `{delete | destroy | list | flush} chain [family] table-name chain-name`
  - `rename chain [family] table-name chain-name new-name`
- Rules
  - `{add | insert} rule [family] table-name chain-name [handle rule-handle] stmt`
    - `add` appends the rule to the chain, or after the specified rule handle
    - `insert` prepends the rule to the chain, or before the specified rule handle
    - it can also nest under `chain {...}`
      - `add`, `rule`, `famliy`, `table-name`, and `chain-name` are always implied
    - `stmt` consists of expressions and statements
  - `{delete |destroy | reset} rule [family] table-name chain-name handle rule-handle`
    - `reset` resets stateful objects
- Sets
  - `add set [family] table-name set-name {...}` creates a set
    - it can also nest under `table {...}`
      - `add`, `family`, and `table-name` are always implied
  - `{delete | destroy | list | flush | reset} set [family] table-name set-name`
- Maps
  - `add map [family] table-name map-name {...}` creates a map
    - it can also nest under `table {...}`
      - `add`, `family`, and `table-name` are always implied
  - `{delete | destroy | list | flush | reset} map [family] table-name map-name`
- Elements
  - `{add | create | delete | destroy | get | reset } element [family] table-name set-name {...}`
    - it works with maps and sets
    - internally, there are only sets; maps are sets whose elements are pairs
- Flowtables
  - `{add | create} flowtable [family] table-name flowtable-name {...}`
    creates a flowtable
  - `{delete | destroy | list} flowtable [family] table-name flowtable-name`
- Listing
- Stateful Objects
  - `{add | delete | destroy | list} {counter | quota | limit} [family] table-name object-name`
    - `counter` consists of two u64 to count packet count and packet size
    - `quota` consists of a u64
    - `limit` is for rate limiting
    - there are also `ct helper`, `ct timeout`, `ct expectation`
  - `{delete | destroy} {counter | quota | limit} [family] table-name handle object-handle`
  - `reset {counter | quota} [family] table-name object-name`
- Expressions
  - each expression has a value and a data type
  - a complex regression can be formed from binary, logical, relational, and
    other types of expressions
- Data Types
  - int, bitmask, string, bool, various addr types, icmp types, etc.
- Primary Expressions
  - a primary expr is a const or refers to a datum from packet's payload,
    metadata, etc.
  - `meta`, `socket`, `osf`, `fib`, `ipsec`, `numgen`, hash
- Payload Expressions
  - a payload expr is a primary expr referring to a datum from packet's payload
  - layer 2: `ether`, `vlan`, `arp`
  - layer 3: `ip`, `icmp`, `igmp`, `ip6`, `icmpv6`
  - layer 4+: `tcp`, `udp`, `udplite`, `sctp`, `dccp`
  - `ah`, `esp`, `comp`, `gre`, `geneve`, `gretap`, `vxlan`, raw, ext hdr
  - `ct` (ct is not a pyalod...)
- Statements
  - it seems rules are evalutated from left to right, with implicit relational
    and logical expressions
    - `tcp dport 80 iifname eth0` has two implicit `==` and one implicit `&&`
  - a stmt is an action, and is either terminal or non-terminal
  - verdict: `accept`, `drop`, `queue`, `continue`, `return`, `jump`, `goto`
  - payload: `payload-expr set`
  - ext header: `ext-hdr-expr set`
  - log: `log`
  - reject: `reject`
  - counter: `counter`
  - conntrack: `ct-expr set`
  - notrack: `notrack`
  - meta: `meta-expr set`
  - limit: `limit`
  - nat: `snat`, `dnat`, `masquerade`, `redirect`
  - tproxy: `tproxy`
  - synproxy: `synproxy`
  - flow: `flow`
  - queue: `queue`
  - dup: `dup`
  - fwd: `fwd`
  - set: `{add|update} @set {expr}`
    - this works with sets and maps
    - the set/map should have been created with
      - `timeout` to remove element automatically
      - `size` to cap max element count
    - when an object is already in the set, `add` is nop while `update`
      refreshes the timeout
  - map: `expr map { elems }`
    - this performs a map lookup
  - vmap: `expr vmap { elems }`
    - this performs a vmap lookup
  - xt: `xt`
- Additional Commands
  - `monitor`

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
    - it uses the token bucket algorithm
      - the bucket capacity is given by `burst`, which is 5 by default
      - the bucket refills tokens at the specified rate
      - a packet matches when there are sufficient tokens in the bucket to be
        taken by the packet
    - if `over` is specified, it matches a packet after the limit is exceeded
      - that is, a packet matches when there are insufficent tokens in the
        bucket
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

## Ban with Rate Limiting

- e.g.,

    table ip ssh-ratelimit {
        set track {
            type ipv4_addr
            flags dynamic
            timeout 30s
        }
        set ban {
            type ipv4_addr
            flags dynamic
            timeout 5m
        }
        chain ratelimit {
            type filter hook input priority filter + 1; policy accept;
            ct state new tcp dport 22 \
              update @track { ip saddr limit rate over 4/minute burst 3 packets } \
              update @ban { ip saddr } \
              drop
            ip saddr @ban drop
        }
    }
- what it does is,
  - for each new connection to tcp port 22, rate limit on its `saddr`
  - if `saddr` makes more than 4 connections per minute on average, add to
    `ban` for 5 minutes
    - if `saddr` makes no connection in 30s, remove rate limit tracking for it
  - else accept by policy

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

## Advanced Data Structures

- <https://wiki.nftables.org/wiki-nftables/index.php/Counters>
  - `counter foo {}` defines a counter to count both 64-bit packets and bytes
  - `tcp dport 80 counter name foo` increments the counter for each packet for
    port 80
  - `nft list counters` lists all counters
  - `nft reset counters` resets all counters
- <https://wiki.nftables.org/wiki-nftables/index.php/Limits>
  - `limit foo { rate 10/second }` defines a limit
    - it matches a packet when the rate is less than 10 packets per second
  - `tcp dport 80 limit name foo accept` accepts a packet when it is matched
    by the limit
  - `nft list limits` lists all limits
  - `nft reset limits` resets all limits
- <https://wiki.nftables.org/wiki-nftables/index.php/Sets>
  - `nft add set <family> <table> foo '{type ipv4_addr;}'` defines a set of
    `ipv4_addr`
  - `nft add element <family> <table> foo { 192.168.0.10 }` adds an ipv4 addr
  - `ip saddr @foo accept` accepts a packet when saddr is in the set
   - set specification
    - `type`
      - `ipv4_addr` for ipv4 addr
      - `ipv6_addr` for ipv6 addr
      - `ether_addr` for ether addr
      - `inet_proto` for inet proto (tcp, udp, icmp, etc.)
      - `inet_service` for inet port (tcp 80, udp 67, etc.)
      - `mark` for mark
      - `ifname` for ifname
    - `timeout` specifies timeout of elemnets
    - `flags`
      - `constant` means the set is immutable
      - `interval` means elements can be intervals
      - `timeout` means there can be per-element timeout
    - `counter` enables per-element counter
- <https://wiki.nftables.org/wiki-nftables/index.php/Maps>
  - internally, a map is a set where each element is a pair
- <https://wiki.nftables.org/wiki-nftables/index.php/Verdict_Maps_(vmaps)>
  - a vmap is a map whose value type is verdict
  - valid verdicts are
    - `accept`
    - `drop`
    - `queue`
    - `continue`
    - `return`
    - `jump <chain>`
    - `goto <chain>`

## Sets

- set parsing
  - `add ...` is matched by `base_cmd`
  - `add set ...` is matched by `add_cmd`
    - `set_spec` is the set name
    - `set_block_alloc` allocs a `set`
    - `set_block` inits `set->key`, `set->flags`, `set->timeout`, etc.
      - flags are parsed to
        - `NFT_SET_CONSTANT`
        - `NFT_SET_INTERVAL`
        - `NFT_SET_TIMEOUT`
        - `NFT_SET_EVAL` (dynamic)
    - `cmd_alloc` allocs a `CMD_ADD`/`CMD_OBJ_SET` cmd
- set evaluation
  - `cmd_evaluate` evaludates the add set cmd
    - `cmd_evaluate_add`
    - `set_evaluate`
- rule parsing
  - `{add|update} @foo {...}` is matched by `set_stmt`
    - the op is `NFT_DYNSET_OP_ADD` or `NFT_DYNSET_OP_UPDATE`
    - `set_stmt_alloc` allocs a `stmt` with `set_stmt_ops`
- rule evaluation
  - `stmt_evaluate` calls `stmt_evaluate_set`
- kernel
  - `nft_dynset_eval` evaluates the dynamic add/update
  - if `NFT_DYNSET_OP_UPDATE`, it refreshes the timeout

# `nft list sets`

- `nft_ctx_new` opens a socket for `NETLINK_NETFILTER`
- `nft_run_cmd_from_buffer` runs `list sets`
  - `nft_parse_bison_buffer` parses `list sets` to a cmd
    - `op` is `CMD_LIST`
    - `obj` is `CMD_OBJ_SETS`
  - `nft_evaluate` evaluates the cmd
    - `nft_cache_evaluate` peeks the cmd and decides what to cache
      - `NFT_CACHE_TABLE` caches tables
      - `NFT_CACHE_SET` caches sets
      - `NFT_CACHE_SETELEM` caches set elements
      - `NFT_CACHE_REFRESH` refreshes the cache
    - `nft_cache_update` updates the cache based on the cache flags
      - `cache_init_tables` calls `mnl_nft_table_dump` to dump all tables and
        caches the tables
      - `cache_init_objects`
        - `set_cache_dump` calls `mnl_nft_set_dump` to dump all sets
        - it then loops through all cached tables and
          - `set_cache_init` caches the sets for each table
          - `netlink_list_setelems` calls `mnl_nft_setelem_get` to dump all
            set elements for each set and caches the elements
    - `cmd_evaluate` calls `cmd_evaluate_list` to evaluate the list cmd
      - generally, evaluation updates the cache according to the cmd
      - but this list cmd is nop because everything is already in cache
    - `nft_netlink` calls `do_command` to do the list cmd
      - generally, `do_command` builds an `nlmsghdr` to be sent to the kernel
      - but this list cmd does not need to communicate to the kernel, and just
        prints the sets from the cache
