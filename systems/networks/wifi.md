WiFi
====

## Standards

- due to demands, products tend to ship before standardization
- 802.11
  - 1997, unofficially Wi-Fi 0
  - Frequency: 2.4 GHz
  - Stream data rate: 1/2 MBits/s
- 802.11b
  - 1999, unofficially Wi-Fi 1
  - Frequency: 2.4 GHz
  - Bandwidth: 20 MHz
  - Stream data rate: 1/2/5.5/11 MBits/s
- 802.11a
  - 1999, unofficially Wi-Fi 2
  - Frequency: 5 GHz
  - Bandwidth: 20 MHz
  - Stream data rate: 6/9/12/18/24/36/48/54 MBit/s
- 802.11g
  - 2003, unofficially Wi-Fi 3
  - Frequency: 2.4 GHz
  - Bandwidth: 20 MHz
  - Stream data rate: 6/9/12/28/24/36/48/54 MBit/s
- 802.11n
  - 2008, Wi-Fi 4
  - Frequency: 2.4/5 GHz
  - Bandwidth: 20/40 MHz
  - Stream data rate: up to 288.8/600 MBit/s
  - MIMO: up to 4 (4x4:4, where the numbers stand for tx, rx, and streams)
- 802.11ac
  - 2014, Wi-Fi 5
  - Frequency: 5 GHz
  - Bandwidth: 20/40/80/160 MHz
  - Stream data rate: up to 346.8/800/1733.2/3466.8 MBit/s
  - MIMO: up to 8
- 802.11ax
  - 2020, Wi-Fi 6
  - Frequency: 2.4/5 GHz
  - Bandwidth: 20/40/80/160 MHz
  - Stream data rate: up to 1147/2294/4804/9608 MBit/s
  - MU-MIMO: up to 8
  - OFDMA
  - 2021, Wi-Fi 6E (6 GHz)
- 802.11be
  - 2024, Wi-Fi 7
  - Frequency: 2.4/5/6 GHz
  - Bandwidth: 20/40/80/160/320 MHz
  - Stream data rate: up to 46120 MBit/s
    - the math goes like
    - base: 7
    - shorter GI: 9
    - better encoding: 172
    - 320MHz channel: 2882
    - 16 streams: 46120
  - MU-MIMO: up to 16
  - OFDMA

## Bitrate Calculation

- how is data put on radio waves?
  - modulations such as AM (amplitude modulation) and FM (frequency
    modulation)
- 802.11g uses OFDM
  - within the 20 MHz bandwidth, there are 64 subcarriers
  - of the 64 subcarriers, 52 are transmitting signals
  - of the 52 subcarriers, 48 are for data
  - symbol duration is 4us (including a guard interval of 0.8us)
- symbol rate
  - there are 0.25M symbols per second (1s didided by 4us) per subcarrier
  - there are 12M symbols per second (0.25M * 48) in total
- 64-QAM has 6 bits per symbol (log2(64))
- code rate (k/n)
  - for every n bits, only k bits are useful
- with code rate 3/4, the maximum bitrate is 54 MBit/s (12M * 6 * 3/4)
- 802.11g bitrate vary depending on
  - bandwidth
  - modulation
  - code rate
- let's do the math again for 802.11ac 
  - within the 160 MHz bandwidth, there are 512 subcarriers
  - of the 512 subcarriers, 484 are transmitting signals
  - of the 484 subcarriers, 468 are for data
  - symbol duration 3.6us (including a guard internal of 0.4us)
  - 256-QAM => 8 bits per symbol
  - code rate 5/6
  - 4 spatial streams
  - 468 / 3.6 * 8 * 5 / 6 * 4 = 3466.7
- what are the differences between 802.11g and 802.11ac?
  - higher bandwidth meaning more subcarriers
  - shorter guard internal meaning higher symbol rate
  - 256-QAM meaning more bits per symbol
  - spatial streams

## Considerations

- 2.4/5 GHz
  - higher frequency penetrates object worse
    - 2.4GHz has better coverage
  - but other technologies use 2.4GHz as well
    - 2.4GHz is more likely to be interfered
- Channels
  - 2.4 GHz has 14 channels, 5MHz apart from each other
    - channel 1 is centered at 2412MHz (+/- 11MHz)
    - channel 2 is centered at 2417MHz (+/- 11MHz)
    - channel 3 is centered at 2422MHz (+/- 11MHz)
    - ...
    - channel 1, 6, and 11 are normally used to avoid overlaps
      - decided by AP
  - 5 GHz has 196 channels, 5MHz apart from each other
    - channel 1 is centered 5005MHz at (+/- 10MHz)
    - channel 2 is centered 5010MHz at (+/- 10MHz)
    - ...
    - to reach 40/80/160 MHz, contiguous channels are combined
      - with higher chance of interference
  - regularations decide which channels are usable in both bands
    - device also supports only a handful of the channels
- multiple devices
  - when multiple devices use the same channel, AP talks to them in turn and
    the bandwidth of the channel is shared among those devices
  - when multiple devices use different channels, all devices get full
    bandwidth?
    - I guess it also depends on the horsepower of the AP?
  - there is AC, advertised speed
    - AC1200: 300Mbit/s for 2.4GHz + 900Mbit/s for 5GHz
    - it looks like two devices on different channels of the same band still
      share the bandwidth.  Perhaps it is not even possible for two devices to
      be on different channels?
- MIMO
  - multiple antennas to exploit multipath propagation
  - written NxM indicating N transmitters and M receivers
  - gives N (or M) times of the bandwidth
- In practise, the best AP in 2024 supports
  - tri-band
    - 802.11be for 6GHz
    - 802.11be for 5GHz
    - 802.11ax for 2.4GHz
    - if mesh, a fourth band for backhaul
  - MU-MIMO: 4x4, 16 streams
    - although clients are commonly limited to 2x2
  - OFDMA: reduce latency
  - beamforming: focus signal to connected clients
  - DFS: avoid channels with high inference

## Roaming

- when two APs have the same SSID and credential, clients can roam from one to
  another
  - the APs should be in the same network.  They should be in AP mode (to
    provide wireless connections to the same network) rather than in router
    mode (to provide wireless connections to their own subnets)
- clients handle roaming
  - APs do not necessarily need to do anything
- fast/seamless roaming
  - 802.11k
    - clients can send Neighbor Report requests to the current AP to
      understand the RF environment, reducing the time spent on scanning
  - 802.11v
    - APs can additionally direct clients to the best one in response to
      Neighbor Report requests
  - 802.11r
    - it reduces the time for wireless auth of WPA2-PSK and WPA2-Enterprise
  - all three require both APs' and clients' support
- wired backhaul
  - APs are connected together using ethernet
- wireless backhaul (mesh)
  - benefits
    - flexible: no cabling required
    - self-forming: each AP in the mesh can automatically calculate the
      possible paths to the router and pick the best one
    - self-healing: each AP in the mesh can switch the path when one fails
  - 802.11s is an open standard

## `iw`

- nl80211
  - `iw features` shows nl80211 features
  - `iw commands` list nl80211 commands
  - `iw monitor` monitors nl80211 events
- regulatory database
  - `iw reg <set|get|reload>` sets/gets/reloads the regulartory database
- phy
  - `iw phy` or `iw list` lists all phys and all their info
  - `iw phy phy0 channels` shows channel info
  - `iw phy phy0 coalesce show` shows coalesce status
  - `iw phy phy0 get txq` shows txq params
  - `iw phy phy0 info` shows all info
  - `iw phy phy0 interface add` adds an interface
    - there is a hw limit on interface types and combinations
    - each interface is assigned a wdev index
      - interfaces of certain interface types (e.g., `managed`) are also
        assigned netdevs
    - by default, there is an interface of type `P2P-device` and an interface
      of type `managed`
  - `iw phy phy0 reg get` shows regulatory domain info
  - `iw phy phy0 wowlan show` shows WoWLAN status
- interface
  - `iw dev <name>...` can be replaced by `iw wdev <idx>...`
  - `iw dev` lists all interfaces of all phys and their info
  - `it dev <name> ftm get_stats` gets FTM responder stats
  - `it dev <name> get power_save` shows powersave status
  - `it dev <name> info` shows the info of an interface
  - `it dev <name> link` shows the info of the current link
  - `it dev <name> mesh_param dump` shows supported mesh params
  - `it dev <name> mpath dump` shows known mesh paths
  - `it dev <name> mpp dump` shows known mesh proxy paths
  - `it dev <name> scan dump -u` shows the current scan results
  - `it dev <name> station dump` shows the station (ap) info
  - `it dev <name> survey dump --radio` shows gathered channel survey data
  - `iw wdev <idx> p2p <start|stop>` starts/stops p2p
- `iw phy` shows the device capabilities
  - `max # scan SSIDs: 20` means a scan that detects up to 20 SSIDs
  - `Device supports RSN-IBSS.` means Robust Security Network protocols and
    Independent Basic Service Set mode
  - `Device supports AP-side u-APSD.` means unscheduled Automatic Power Save
    Delivery, allowing the ap to put the device to power saving mode
  - `Device supports T-DLS.` means Tunneled Direct Link Setup (802.11z)
  - `Available Antennas: TX 0x3 RX 0x3` means 2 TX and 2 RX antennas
  - `Supported interface modes:`
    - `IBSS` is ad-hoc
    - `managed` is the standard wifi client
    - `AP` functions as ap
  - `Band x:`
    - there can be up to 3 bands at 2.4GHz, 5GHz, and 6GHz
    - wifi standards
      - non-HT: 802.11g, wifi 3
      - HT: high throughput, 802.11n, wifi 4
      - VHT: very high throughput, 802.11ac, wifi 5
      - HE: high efficiency, 802.11ax, wifi 6
      - EHT: extremely high throughput, 802.11be, wifi 7
    - 802.11n
      - `Capabilities: 0x19ef`
        - `HT20/HT40` high-throughput 20MHz and 40MHz channels
        - `RX HT20 SGI` short GI at 20MHz
        - `RX HT40 SGI` short GI at 40MHz
      - `HT Max RX data rate: 300 Mbps`
        - MCS 15 allows 300Mbps at 40MHz and short GI
      - `HT TX/RX MCS rate indexes supported: 0-15`
        - MCS 15 means 2 spatial streams, 64-QAM, 5/6 coding rate
    - 802.11ac
      - `VHT Capabilities (0x039071f6):` for VHT (very high-throughput,
        802.11ac)
        - `Supported Channel Width: 160 MHz` for 160MHz channels
        - `short GI (80 MHz)`
        - `short GI (160/80+80 MHz)`
      - `VHT RX MCS set:` lists supported RX MCS indices
        - MCS 9 means 256-QAM, coding rate 5/6
          - 400Mbps with 2 streams, 40MHz, and short GI
      - `VHT TX MCS set:` lists supported TX MCS indices
    - 802.11ax
      - `HE Iftypes: managed`
      - `HE MAC Capabilities (0x78018a30abc0):` lists mac caps
      - `HE PHY Capabilities: (0x0c3f0e09fd098c160ff001):` lists phy caps
        - `HE40/2.4GHz` supports 40MHz at 2.4GHz
        - `HE40/HE80/5GHz` supports 40MHz and 80MHz at 5GHz
        - `HE160/5GHz` supports 160MHz at 5GHz
      - `HE RX MCS and NSS set <= 80 MHz` lists supported RX MCS indices
        - MCS 11 means 1024-QAM, coding rate 5/6
          - 574Mbps with 2 streams, 40MHz, and short GI
      - `HE TX MCS and NSS set <= 80 MHz` lists supported TX MCS indices
    - 802.11be
      - `EHT Iftypes: managed`
    - 802.11g
      - `Bitrates (non-HT):` lists supported non-HT bitrates
    - `Frequencies:` lists supported channels
  - `Supported commands:` are commands supported by device
  - `WoWLAN support:` is WoWLAN support
  - `valid interface combinations:` gives the valid interface combinations
  - `HT Capability overrides:` lists HT params that can be overriden
  - `Device supports foo` for various features
  - `max # scan plans: 2`, a scan plan refers to a predefined set of scanning
    parameters
  - `Supported extended features:` for various features
  - non-HT (non-high-throughput): 802.11g
  - HT (high-throughput): 802.11n
  - VHT (very-high-throughput): 802.11ac
  - MCS: modulation and and coding scheme
    - 8: 256-QAM 3/4
    - 9: 256-QAM 5/6
- `iw dev <dev> station dump` shows the AP capabilities
- `iw dev <dev> link` shows the current link
  - `freq: 5180`
    - channel 36 (`(5180-5000)/5`)
  - `rx bitrate:` varies depending on signal strengths and other factors
    - walking around with my laptop, I've seen
    - `rx bitrate: 24.0 MBit/s`
      - this is 802.11g
    - `rx bitrate: 51.6 MBit/s 40MHz HE-MCS 2 HE-NSS 1 HE-GI 0 HE-DCM 0`
      - `40MHz` is channel width
      - `HE` is 802.11ax
      - `MCS 2` is QPSK, 3/4
      - `NSS 1` is 1 spatial stream
      - `GI 0` is short (800ns) GI
      - `DCM 0` disables DCM
        - DCM halves the bandwidth for stronger signal
    - `rx bitrate: 68.8 MBit/s 40MHz HE-MCS 1 HE-NSS 2 HE-GI 0 HE-DCM 0`
      - `MCS 1` is QPSK, 1/2
      - `NSS 2` is 2 spatial stream
    - `rx bitrate: 68.8 MBit/s 40MHz HE-MCS 3 HE-NSS 1 HE-GI 0 HE-DCM 0`
      - `MCS 3` is 16-QAM, 1/2
    - `rx bitrate: 103.2 MBit/s 40MHz HE-MCS 2 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 103.2 MBit/s 40MHz HE-MCS 4 HE-NSS 1 HE-GI 0 HE-DCM 0`
      - `MCS 4` is 16-QAM, 3/4
    - `rx bitrate: 309.7 MBit/s 40MHz HE-MCS 6 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 344.1 MBit/s 40MHz HE-MCS 7 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 413.0 MBit/s 40MHz HE-MCS 8 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 458.8 MBit/s 40MHz HE-MCS 9 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 516.0 MBit/s 40MHz HE-MCS 10 HE-NSS 2 HE-GI 0 HE-DCM 0`
    - `rx bitrate: 573.5 MBit/s 40MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0`
      - `MCS 6` is 64-QAM, 3/4
      - `MCS 7` is 64-QAM, 5/6
      - `MCS 8` is 256-QAM, 3/4
      - `MCS 9` is 256-QAM, 5/6
      - `MCS 10` is 1024-QAM, 3/4
      - `MCS 11` is 1024-QAM, 5/6
    - when using 802.11ac
      - `VHT` is 802.11ac
      - `short GI` is 400ns GI

## `iwd`

- quick connect
  - `systemctl enable --now iwd`
  - `iwctl`
    - `device list`
    - `station <dev> scan`
    - `station <dev> get-networks`
    - `station <dev> connect "<ssid>"`
      - password will be prompted
  - the link and password will be saved to `/var/lib/iwd`

## `wpa_supplicant`

- quick connect
  - create `/etc/wpa_supplicant/wpa_supplicant-<iface>.conf`

      ctrl_interface=DIR=/run/wpa_supplicant
      network={
        ssid="<ssid>"
        psk="<password>"
      }
  - `chmod 600`
  - `systemctl enable --now wpa_supplicant@<iface>`
- operation modes
  - `wpa_supplicant` can be started with `-u`
    - this enables the dbus service
    - network-manager uses the dbus service to add and config interfaces
  - `wpa_supplicant` can be started with `-i <dev>` and
    `-C /run/wpa_supplicant`
    - this enables `/run/wpa_supplicant/<dev>` socket
    - `wpa_cli` uses the socket to config `<dev>`
  - `wpa_supplicant` can be started with `-i <dev>` and
    `-c /etc/wpa_supplicant/wpa_supplicant-<dev>.conf`
    - the config file can optionall have
      `ctrl_interface=DIR=/run/wpa_supplicant` to enable the control socket
      for `wpa_cli`
- `wpa_cli`
  - `interface` lists interfaces or selects interface
  - `status` shows current status
  - `signal_poll` shows current signal params
  - `scan` requests a BSS scan
  - `scan_results` shows the latest scan results
  - `list_networks` lists configured networks
  - `add_network` adds a new network
  - `remove_network` removes a network
  - `set_network <id> <key> <val>` sets a network var
    - `set_network` lists available network vars, such as `ssid` and `psk`
  - `select_network` selects a network and disables others

## My APs

- Google Wifi
  - qcom ipq4019 (A7 x4 @717Mhz), 512MB ram, 4GB emmc
  - wifi 5 (802.11ac)
  - AC1200
    - 300Mbps at 2.4GHz: MCS 7 (64-QAM), 400ns GI, 40MHz ch, 2x2 MIMO
    - 900Mbps at 5GHz: MCS 9 (256-QAM), 400ns GI, 80MHz ch, 2x2 MIMO
- TP-Link EAP610
  - qcom ipq6000 (A53 x4 @1.2GHz), 256MB ram, 128MB flash
  - wifi 6 (802.11ax): qcom qcn5022, qorvo qpf4588, skyworks sky85340-11
  - 1gb ethernet: realtek RTL8211F
  - AX1800
    - 574Mbps at 2.4GHz: MCS 11 (1024-QAM), 800ns GI, 40MHz ch, 2x2 MIMO
    - 1201Mbps at 5GHz: MCS 11 (1024-QAM), 800ns GI, 80MHz ch, 2x2 MIMO
- on a client with with BCM4352
  - 130Mbps at 2.4GHz: MCS 7 (64-QAM), 800ns GI, 20MHz ch, 2x2 MIMO
  - 270Mbps at 5GHz: MCS 7 (64-QAM), 800ns GI, 40MHz ch, 2x2 MIMO
  - looks like,
    - the ap picks a 20MHz ch at 2.4GHz and a 40MHz ch at 5.GHz (due to
      inferences)
    - the client is only capable of MCS 7 and 800ns GI
