unbound
=======

## DNS

- authoritative name servers
  - an authoritative name server is authoritative for its zone
  - `named.root` has `NS` and `A` records for authoritative name servers of `.`
    - <https://www.internic.net/domain/named.root>
    - they are also known as root name servers
  - root name servers have `NS` and `A` records for the authoritative name
    servers of `<tld>.`
  - authoritative name servers of `<tld>.` have `NS` and `A` records for
    authoritative name servers of `<domain>.<tld>.`
  - and so on
- recursive name servers
  - they are also known as recursive DNS resolvers
  - they answer queries for `<subdomain>.<domain>.<tld>.` by
    - consult `named.root` for the ip of root name servers
    - consult root name servers for ip of authoritative name servers of
      `<tld>.`
    - consult authoritative name servers of `<tld>.` for ip of authoritative
      name servers of `<domain>.<tld>.`
    - consult authoritative name servers of `<domain>.tld>.` for ip of
      `<subdomain>.<domain>.<tld>.`
- after buying `<domain>.<tld>.` from a registrar and specifying authoritative
  name servers of the zone, the registrar updates the `NS` and `A` records of
  `<domain>.<tld>.` from authoritative name servers of `<tld>.`
- `unbound` implements a recursive name server
  - in contrast, `bind` or `nsd` implements an authoritative name server

## TLDs

- <https://en.wikipedia.org/wiki/Top-level_domain>
  - <https://www.iana.org/domains/root/db>
- ARPA, infrastructure
  - `.arpa`
    - `in-addr.arpa.` for ipv4 reverse-mapping
    - `ip6.arpa.` for ipv6 reverse-mapping
- gTLD, generic
  - three or more letters
- grTLD, restricted generic
  - `.biz`
  - `.name`
  - `.pro`
- sTLD, sponsored
  - `.aero`
  - `.asia`
  - `.coop`
  - `.edu`
  - `.gov`
  - `.int`
  - `.jobs`
  - `.mil`
  - `.post`
- ccTLD, country code
  - ISO 3166 two-letter country codes
- tTLD, test
- special-use domain names
  - <https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml>
  - `home.arpa.` is for residential private network
  - `example.`
  - `example.com.`
  - `example.net.`
  - `example.org.`
  - `invalid.` is always invalid
  - `local.` is for mdns
  - `localhost.` is for local private network
  - `onion.` is for tor
  - `test.` is for testing
- RFC6762 lists some non-standard tlds for private networks
  - `intranet.`
  - `internal.`
    - this is official now with ICANN resolution 2024.07.29.06
  - `private.`
  - `corp.`
  - `home.`
  - `lan.`
  - redhat uses `localdomain.`
  - openwrt uses `lan.`

## Config

- `interface: 0.0.0.0` listens on all ipv4
- `access-control: 192.168.86.0/24 allow` allows a subset
- `private-address: 192.168.86.0/24` filters out all answers in the subnet...
- `private-domain: "internal."` ...unless the domain is also private
- `local-zone: "internal." static` defines a local zone
  - `local-data:` defines a record
- `local-zone: "86.168.192.in-addr.arpa." static`
  - `local-data-ptr:` defines a PTR record
- `local-zone: "bad" always_null` blocks a bad domain
- zones
  - `forward-zone:` defines a zone where all queries are forwarded to another
    name server
    - unbound becomes a stub resolver for the zone
  - `auth-zone:` defines a zone for which unbound is authoritative
    - when unbound receives a query for the zone, in this order,
      - it consults the zone locally if `for-downstream: yes`
      - it consults the cache
      - it consults the zone locally if `for-upstream: yes`
      - it consults the zone's real authoritative name servers
  - `stub-zone:` defines a zone with private authoritative name servers
    - unbound uses the specified authoritative name servers for the zone
      directly, rather than finding them with recursive queries
    - this is used for private zones with designated authoritative name
      servers
  - `local-zone: <zone> <type>` defines a local zone for which unbound is
    authoritative
    - `local-data` defines a record for the zone
      - each record has a name (e.g., `www.example.com`) and a type (e.g.,
        `A`)
      - a `local-data` that is not in any `local-zone` is assumed to be in a
        `transparent` local zone
    - `type` specifies the policy when no `local-data` matches a query
      - `deny` drops the query silently
      - `refuse` responds `REFUSED`
      - `static` responds `NXDOMAIN` (no matching name) or `NODATA` (has
        matching name but different record type)
      - `transparent` continues with the regular recursive query if no
        matching name
      - `typetransparent` continues with the regular recursive query if no
        matching name and record type
      - `redirect` always responds the single `local-data` of the local zone
    - unless overriden by explicit `local-zone`, unbound defines these local
      zones implicitly
      - `local-zone: "localhost." redirect`
      - `local-zone: "127.in-addr.arpa." static`
      - `local-zone: "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa." static`
      - `local-zone: "home.arpa." static`
      - `local-zone: "resolver.arpa." static`
      - `local-zone: "service.arpa." static`
      - `local-zone: "onion." static`
      - `local-zone: "test." static`
      - `local-zone: "invalid." static`
      - more

## Zone File

- `com.` zone file
  - `example.com NS ns.foo.net`
    - this is delegation record, specifying authoritative name server of
      `example.com`
  - `ns.foo.net A ...`
    - this is glue record, sepcifying the ip of the authoritative name server
      of `example.com`
  - registrar gets the info from the domain owner and provides it to tld name
    servers
- `example.com.` zone file
  - `example.com. 86400 IN SOA ns.foo.net. contact.bar.org. 1 21600 3600 604800 86400`
    - this provides administrative info about the zone
    - first `86400` is TTL
    - `ns.foo.net.` is MNAME, the primary authoritative name server
    - `contact.bar.org.` is RNAME, the contact email address
    - `1` is SERIAL; when incremented, secondary authoritative name servers
      should sync their zone files from the primary authoritative name server
    - `21600` is REFRESH, the period secondary authoritative name servers
      should use to check the SOA record
    - `3600` is RETRY, the period secondary authoritative name servers should
      wait before checking the SOA record again on errors
    - `604800` is EXPIRE, how long secondary authoritative name servers should
      wait before expiring the synced zone files after the last successful
      sync
    - second `86400` MINIMUM, to cap TTL for negative response
  - `example.com. 86400 IN NS ns.foo.net.`
    - this specifies authoritative name servers of the zone
    - this is the authoritative answer, if it ever disagrees with `com.` zone
      file
