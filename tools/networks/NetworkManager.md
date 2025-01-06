NetworkManager
==============

## Overview

- NetworkManager uses `/etc/NetworkManager/NetworkManager.conf`
  - don't edit
  - use `/etc/NetworkManager/conf.d` instead
- `udev` is used for device discovery
- for any network state change, `/etc/NetworkManager/dispatcher.d` is invoked
- handy tools
  - nmcli
  - nm-connection-editor
- to enable,
  - `systemctl enable --now NetworkManager`

## nmcli

- to scan APs,
  - `nmcli device wifi list`
- to connect to AP,
  - `nmcli wifi connect <ssid>`
  - it prompts password on one machine but requires `password` on another
- to enable/disable wifi,
  - `nmcli radio wifi on` or `off`
- to manage/unmanage a device,
  - `nmcli device set <dev> managed yes/no`

## Connection Profiles

- NetworkManager is based on connection profiles, or simply connections
- connection profiles can be from several sources
  - `ifupdown` plugin parses `/etc/network/interfaces` to create profiles
  - `keyfile` plugin parses keyfiles from various locations
    - `/etc/NetworkManager/system-connections`
    - `/usr/lib/NetworkManager/system-connections`
    - `/run/NetworkManager/system-connections`
  - `nm-applet` and `nm-connection-editor` can interactively create
    connections profiles
- once a connnection profile is created, it is saved to disk using `keyfile`
  plugin
- A connection has tons of properties.  Most of the properties use default
  values or use values defined by `[connection]` in `NetworkManager.conf`
  - `autoconnect` specifies whether to auto connect when resources are
    available
    - e.g., the ap shows up and there is a wireless device
    - e.g., any wired device is connected
  - `ipv4.method` specifies how to set up IPv4 on this connection.  It is
    normally `auto` which does DHCP.  It can also be `shared`, which means the
    connection has no upstream network and behaves as a gateway to downstream.
    NM then picks an IP, starts dnsmasq for downstream DHCP and DNS, and sets
    up NAT.
- use `nm-connection-editor` to see/edit connections

## NetworkManager.conf

- `[main]`
  - `dhcp=` can use dhclient, dhcpcd, or internal for built-in DHCP client
  - `dns=` can be default, dnsmasq, systemd-resolved, or others
    - default updates `/etc/resolv.conf` using `resolvconf`
    - dnsmasq starts dnsmasq at localhost and points `/etc/resolve.conf` to
      localhost
    - systemd-resolved uses systemd-resolved
    - both `dnsmasq` and `systemd-resolved` provide dbus interfaces to receive
      DNS servers and options from NM
      - to see the upstream dns servers of dnsmasq,
        `sudo dbus-send --system --print-reply --dest=org.freedesktop.NetworkManager.dnsmasq /uk/org/thekelleys/dnsmasq org.freedesktop.NetworkManager.dnsmasq.GetServerMetrics`
- `[keyfile]`
  - `path=` specifies the location of keyfiles.  By default,
    `/etc/NetworkManager/system-connections`
  - `unmanaged-devices=` specifies devices that NM should not manage
- `[ifupdown]`
  - `managed=` specifies whether devices listed in `/etc/network/interfaces`
    should be managed by NM
- `[connection]`
  - default values for connnectoins
  - `ipv4.route-metric` defaults to -1
    - for wired, the metric will be 100
    - for wirelss, the metric will be 600
- `[device]`
  - `match-device=` specifies the device
  - `managed=` specifies whether the device is managed or ignored

## nm-applet

- by default, nm-applet calls `gtk_status_icon_new` to create a status icon
  - it internally uses `XEmbed` protocol and is not supported on wayland
- when `--indicator` is specified, or when on wayland since nm-applet 1.32.0,
  it switches to `app_indicator_new`
  - it internally uses `StatusNotifierItem` protocol
