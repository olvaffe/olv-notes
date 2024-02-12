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

- `iw dev <dev> link` shows the current link
  - `freq: 5180`
    - channel 36 (=(5180-5000)/5)
  - rx bitrate: 780.0 MBit/s VHT-MCS 8 80MHz short GI VHT-NSS 2
    - VHT-MCS: Very High Throughput Modulation and Coding Scheme
      - 8: 256-QAM, code rate 3/4
    - 80MHz: bandwidth
      - 4 (=80/20) channels are used
    - short GI: 0.4us guard interval
    - VHT-NSS: number of spatial streams
      - 2: 2 spatial streams
  - tx bitrate: 866.7 MBit/s VHT-MCS 9 80MHz short GI VHT-NSS 2
    - VHT-MCS 9: 256-QAM, code rate 5/6
- `iw phy` shows the device capabilities
  - non-HT (non-high-throughput): 802.11g
  - HT (high-throughput): 802.11n
  - VHT (very-high-throughput): 802.11ac
  - MCS: modulation and and coding scheme
    - 8: 256-QAM 3/4
    - 9: 256-QAM 5/6
- `iw dev <dev> station dump` shows the AP capabilities

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
