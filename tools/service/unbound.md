unbound
=======

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
