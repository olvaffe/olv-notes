Homelab
=======

## Home Network

- modem: ARRIS SURFboard SB8200
  - DOCSIS 3.1
  - 1Gb uplink
  - 1Gb ethernet port x2
- router: Google Wifi
  - AC1200 (300@2.4GHz, 900@5GHz)
  - 1Gb ethernet port x2 (for uplink and downlink)
- switch: TP-Link TL-SG1005P
  - 1Gb port x5
    - 4 of them are poe+
  - max power consumption
    - 4.0W without PD
    - 64.4W with 56W PD
  - power consumption in general
    - 1Gb x8, max 3W
      - if poe x4, add 3W
      - if poe x8, add 6W
      - if smart, add 3W
      - if managed (vlan, 802.1x, etc.), no difference
    - 2.5Gb x8, max 7.5W
    - 10Gb x8, max 30W, active cooling
- ap: TP-Link EAP610 V2
  - AX1800 (574@2.4GHz, 1201@GHz)
- considerations
  - add another ap
  - replace router by fanless homelab
    - consult pfsense netgate for L3 forwarding, firewall, and vpn numbers
    - should i run services on a separate server?

## Distros

- type-1 hypervisor
  - proxmox
    - debian-based
    - x86-only (there is unofficial arm port)
    - upgrade can be done via apt or reinstall
      - apt preverses configs
      - but backup is always recommended
- router
  - openwrt
    - linux-based
    - x86 and arm
    - wired and wireless
    - many packages
    - upgrade wipes the rootfs, flashes the new firmware, and reboots
      - the firmware must be downloaded manually
      - all configs/packages are lost!
      - backup and restore can be easy if customizations are limited to UCI
        (`/etc/config`)
  - pfesnse
    - freebsd-based
    - x86 and arm
    - wired only
    - only routing-related packages
    - upgrade can be triggered from webgui or from console
      - it downloads the update, applies it, and reboots
      - it tries to preserve configs
        - but backup is always recommended
- general purpose
  - armbian
  - debian
    - `unattended-upgrades`

## Credentials

- google app password (gmail)
- duckdns access token
- wireguard private key

## Networking

- types of devices
  - trusted clients (laptops, phones, etc.)
  - untrusted clients (most iot)
  - servers (ssh, cast, etc.)
  - network management (router/firewall, dhcp, dns, omada, etc.)
- WAN
  - ISP is Xfinity
  - need to reboot the modem to get a new DHCP lease
  - modem provides a private network, `192.168.100.0/24`, briefly during boot
  - xfinity provides
    - an ipv4 address in `73.92.126.0/23`
    - two ipv6 addresses (from RA and from DHCPv6 respectively?)
    - a `/60` delegated prefix (it does not include the two ipv6 addresses)
  - `20-wan.network`
    - `DHCP=ipv4` suffices
    - ipv6 ra is enabled by default and will trigger dhcpv6
- LAN
  - `20-lan.network`
    - `Address=192.168.0.254/24`
    - `DHCPServer=yes` to start dhcpv4
    - `IPForward=yes` to forward both ipv4 and ipv6
    - `IPMasquerade=ipv4` for ipv4 nat
    - `DHCPPrefixDelegation=yes` to be assigned an ipv6 subnet
    - `IPv6SendRA=yes` for ipv6 ra
    - `IPv6AcceptRA=no`
- `systemd-resolved`
  - `20-lan.network`
    - `[Network]`
    - `DNS=75.75.75.75 75.75.76.76 2001:558:feed::1 2001:558:feed::2` from
      xfinity
    - `[DHCPServer]`
    - `DNS=_server_address` to advertise itself
    - `[IPv6SendRA]`
    - `EmitDNS=no` to not advertise wan dns server
  - `resolved.conf`
    - `DNSStubListenerExtra=192.168.0.254` to listen on the interface
  - `/etc/hosts` for local host names
- it might be better to use dnsmasq for both dhcp and dns

## Omada Controller

- an omada controller manages omada devices
  - this includes omada routers, switches, and aps
  - devices can be from different sites
  - the controller can discover devices on the same subset
  - for devices on different subsets,
    - one can log in into the devices and point them to the controller
    - or, for aps, one can run the omada discovery utilities on those
      different subsets
  - discovered devices are "pending" and the controller must "adopt" them
- global view
  - there are 3 tabs: dashboard, devices, and logs
  - there are also account and settings (of the controller)
- per-site view
  - there are 9 tabs: dashboard, statistics, map, devices, clients, insights,
    logs, tools, and reports
  - there is also settings (of the site)
    - this is probably the most important tab

## Duck DNS

- <https://www.duckdns.org/>
- `https://www.duckdns.org/update?domains={YOURVALUE}&token={YOURVALUE}[&ip={YOURVALUE}][&ipv6={YOURVALUE}][&verbose=true][&clear=true]`
  - `domains` is a comma-separated list of subdomains to update
  - `token` is the account token
  - `ip` and `ipv6` update ipv4 and ipv6 records respectively
    - if `ipv6` is missing or empty, it updates the ipv4 record
      - if `ip` is missing or empty, it auto-detects the ipv4 addr
    - if `ipv6` is non-empty, it updates the ipv6 record
      - if `ip` is non-empty, it updates the ipv4 record as well
  - `verbose=true` requests verbose responses
  - `clear=true` clears both ipv4 and ipv6 records
- `https://www.duckdns.org/update?domains={YOURVALUE}&token={YOURVALUE}&txt={YOURVALUE}[&verbose=true][&clear=true]`
  - same as above, but updates the `TXT` record
  - this is useful for ACME DNS-01 challenge
- public ip detection
  - `curl [-4|-6] <url>`, where `<url>` is
  - `https://ifconfig.me`
  - `https://icanhazip.com`
  - `https://ipecho.net/plain`

## `gcloud`

- free tier
  - <https://cloud.google.com/free/docs/free-cloud-features#free-tier>
  - compute engine
    - `e2-micro`
    - `us-west1`
    - 30 GB-months standard persistent disk
- <https://cloud.google.com/sdk/docs>
  - untar the tarball
  - `install.sh` or use `google-cloud-sdk/bin/gcloud` directly
- components
  - `gcloud components list`
  - `gcloud components update`
- init
  - `gcloud init`
- auth
  - `gcloud auth list` lists accounts
  - `gcloud auth login` logs in
- compute
  - os login
    - `gcloud compute project-info add-metadata --metadata=enable-oslogin=true`
    - `gcloud compute os-login ssh-keys add --key-file ~/.ssh/id_rsa.pub`
    - the user name is the email with special characters (`@` or `.`) replaced
      by `_`
  - quick ssh
    - `gcloud compute ssh <vm-instance>`
    - this adds the ssh key to the vm temporarily and ssh to it
