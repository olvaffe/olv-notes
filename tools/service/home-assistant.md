# Home Assistant

## Standards

- Zigbee
  - app layer (osi l7)
  - network layer (osi l3)
  - 802.15.4 mac (osi l2)
  - 802.15.4 phy (osi l1)
- Z-Wave
  - app layer (osi l7)
  - network layer (osi l4)
  - transport layer (osi l3)
  - mac layer (osi l2)
  - phy layer (osi l1)
- BLE
  - app layer (osi l7)
    - profiles
    - gatt
    - att
  - l2cap (osi l2)
  - link layer (osi l2)
  - physical layer (osi l1)
- WiFi
  - 802.11 mac (osi l2)
  - 802.11 phy (osi l1)
- Thread
  - network layer (osi l3)
    - ipv6
    - 6LoWPAN
  - 802.15.4 mac (osi l2)
- Matter
  - app layer (osi l7)
  - tcp/udp (osi l4)
  - ipv6 (osi l3)
  - thread/wifi/ble/etc (osi l1-l2)
- in a zigbee mesh network,
  - there is a coordinator (ZC)
    - e.g., home assistant with zigbee usb dongle
  - there are zero or more routers (ZR)
    - wall-powered devices are often both ZR and ZED
    - ZR routes traffic between ZR and ZEDs to increase coverage
  - there are zero or more end devices (ZED)
    - battery-powered devices are often just ZED
- in a thread mesh network,
  - <https://openthread.io/guides/thread-primer/node-roles-and-types>
  - similar to zigbee, there are routers and end devices
    - wall-powered devices are often FTD
    - battery-powered devices are often MTD
  - thread border router (TBR)
    - each node in the thread mesh network has an ipv6 address
    - without TBR, the thread mesh network is a closed ipv6 network
    - with TBR, it can route traffic between the thread mesh network and other
      ipv6 networks (e.g., home wifi network)
  - there is no coordinator?
    - a coordinator is defined by the app layer, such as matter defines a
      controller
    - a matter controller is also often a TBR

## Integration: TP-Link Smart Home

- <https://www.home-assistant.io/integrations/tplink/>
- legacy plugs without DNS-SD
  - discovery: udp port 9999
    - ha broadcasts `get_sysinfo` to udp port 9999
    - plugs respond with `sysinfo` which contains macs, models, etc.
      - plug ips are from the unicast response saddrs
    - there is also dhcp snooping
  - control: tcp port 9999

## Integration: Android TV Remote

- <https://www.home-assistant.io/integrations/androidtv_remote/>
- discovery: dns-sd
  - `_androidtvremote2._tcp.local. IN PTR MyTV._androidtvremote2._tcp.local.`
    - this advertises an instance, `MyTV`, of `_androidtvremote2` service
  - `MyTV._androidtvremote2._tcp.local. IN SRV 0 0 6466 MyTV.local.`
    - this advertises tcp port 6466 of host `MyTV.local`
  - `MyTV.local. IN A 192.168.1.50`
    - this advertises A record of `MyTV.local`
  - `MyTV._androidtvremote2._tcp.local. IN TXT "bt=<mac>"`
    - this advertises MAC address
    - mac is used to uniquely identify the device
    - it can also be used in dhcp snooping to update ip
- pairing: tcp port `6466+1`
  - ha generates a cert
  - ha connects to port 6467 to start pairing
  - tv displays a pin
  - ha gets the pin from user input
  - ha completes pairing
  - tv trusts the cert
- control: tcp port `6466`
  - the protocol supports `RemoteKeyInject` and `RemoteAppLinkLaunchRequest`
  - ha exposes a media player entity and a remote entity

## Integration: Google Cast

- <https://www.home-assistant.io/integrations/cast/>
- discovery: dns-sd
  - `_googlecast._tcp.local. IN PTR Model-UUID._googlecast._tcp.local.`
    - this advertises an instance, `Model-UUID`, of `_googlecast` service
  - `Model-UUID._googlecast._tcp.local. IN SRV 0 0 8009 UUID.local.`
    - this advertises tcp port 8009 of host `UUID.local`
  - `UUID.local. IN A 192.168.1.50`
    - this advertises A record of `UUID.local`
  - `Model-UUID._googlecast._tcp.local. IN TXT "..."`
    - `id` is uuid
    - `cd` is cloud uuid (when added to google home)
    - `rm` is rma id
    - `ve` is protocol version
    - `md` is model name
    - `ic` is path to icon
    - `fn` is friendly name
    - `ca` is caps (video, audio, etc.)
    - `st` is status
    - `bs` is mac
    - `nf` is notification flag (for google home)
    - `rs` is current active app

## Integration: Roborock

- <https://www.home-assistant.io/integrations/roborock/>
- discovery
  - ha snoops dhcp for roborock
  - if mac is unknown, ha prompts user to login to cloud
    - this is to retrieve `duid` (device uid), `local_key` (aes key), `rriot`
      (mqtt cred), etc.
  - ha connects to cloud and uses mqtt (tcp port 8883) for control
  - ha discovers the vacuum and dock local ips
    - it makes `get_network_info` mqtt call
    - it also broadcasts to udp port 58866 locally
- control: tcp port 58867 and mqtt
  - ha subscribes to push notifications using mqtt
  - ha polls detailed states periodically
    - locally but can fall back to mqtt
