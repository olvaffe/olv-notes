WiFi
====

## Standards

- due to demands, products tend to ship before standardization
- 802.11b
  - 1999, unofficially Wi-Fi 1
  - Frequency: 2.4 GHz
  - Bandwidth: 22 MHz
  - Stream data rate: 1/2/5.5/11 MBits/s
  - MIMO: no
- 802.11a
  - 1999, unofficially Wi-Fi 2
  - Frequency: 5 GHz
  - Bandwidth: 5/10/20 MHz
  - Stream data rate: 6/9/12/18/24/36/48/54 MBit/s
  - MIMO: no
- 802.11g
  - 2003, unofficially Wi-Fi 3
  - Frequency: 2.4 GHz
  - Bandwidth: 5/10/20 MHz
  - Stream data rate: up to 13.5/27/54 MBit/s
  - MIMO: no
- 802.11n
  - 2008, Wi-Fi 4
  - Frequency: 2.4/5 GHz
  - Bandwidth: 20/40 MHz
  - Stream data rate: up to 288.8/600 MBit/s
  - MIMO: 4
- 802.11ac
  - 2014, Wi-Fi 5
  - Frequency: 5 GHz
  - Bandwidth: 20/40/80/160 MHz
  - Stream data rate: up to 346.8/800/1733.2/3466.8 MBit/s
  - MIMO: 8
- 802.11ax
  - 2020, Wi-Fi 6
  - 2021, Wi-Fi 6E (6 GHz)
  - Frequency: 2.4/5/6 GHz
  - Bandwidth: 20/40/80/80+80 MHz
  - Stream data rate: up to 1147/2294/4804/9608 MBit/s
  - MIMO: 8
- 802.11be
  - 2024, Wi-Fi 7???

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
- In practise,
  - best AP (in 2020) uses 802.11ac, 4x4, supporting beamforming and all DFS
    channels
  - although clients are commonly limited to 2x2

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
