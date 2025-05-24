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

## Names and Name Resolutions

- POSIX functions
  - `gethostname` uses `uname` syscall to query `utsname` and retruns
    `utsname::nodename`
  - `sethostname` uses `sethostname` syscall to set `utsname::nodename`
  - `getnameinfo`, which obsoletes `gethostbyaddr`, resolves addr to name
  - `getaddrinfo`, which obsoletes `gethostbyname`, resolves name to addr
    - the behavior is configured by `/etc/nsswitch.conf`
- GLIBC Resolver
  - `man resolver` and `man resolv.conf`
  - DNS name resolution
  - `getaddrinfo` can be configured to use the resolver
- POSIX tools
  - `hostname` calls `gethostname` to return the hostname
    - kernel defaults to `CONFIG_DEFAULT_HOSTNAME`, which is usually `(none)`
    - systemd calls `sethostname` with the first of
      - `systemd.hostname=` from kernel cmdline
      - `/etc/hostname`
        - `man 5 hostname` recommends a single label without any dot
        - some apps might break though
      - DHCP lease
      - hard-coded `localhost`
  - `domainname`
    - this is NIS domain name and is irrelevant nowadays
  - `dnsdomainname` and FQDN
    - FQDN is `addrinfo::ai_canonname` queried from `getaddrinfo`
    - the part after the first dot is the DNS domain name
- `getaddrinfo` and `nsswitch.conf`
  - systemd recommendations for name resolution
    - `hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns`
  - `mymachines` resolves with `systemd-machined` for vms/containers
  - `resolve` resolves with `systemd-resolved` but may not be available
    - `!UNAVAIL=return` means, if the return code is not `UNAVAIL`, return
      immediately
  - `files` resolves with `/etc/hosts`
    - systemd recommends a short nickname for `/etc/hostname`, which is not
      used here
    - on the othe hand, both the canonical name as well as the short nickname
      (alias) should be listed in `/etc/hosts`
    - this way, `getaddrinfo` can resolve the short nickname to the canonical
      name, which is the FQDN of the local machine
  - `myhostname` resolves local names
    - it resolves the current hostname to `127.0.0.2`
    - it resolves `localhost`, `localhost.localdomain`, `.localdomain`, and
      `.localhost.localdomain` to `127.0.0.1`
    - it resolves `_gateway` to the current default gateway
    - it resolves `_outbound` to the current ip addr
  - `dns` resolves with DNS configured by `/etc/resolv.conf`
