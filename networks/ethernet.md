Ethernet
========

## Standards

- classic ethernet, 10Mb/s
  - 10BASE5
    - 802.3-1983
    - up to 500m
    - baseband
  - 10BASE2
    - 802.3a-1988
    - up to 200m
  - 10BASE-T
    - 802.3i-1990
    - up to 100m
    - twisted pair
    - Cat 3 / Cat 4
  - there is also 10BASE-FX
    - fiber optic cable
- fast ethernet, 100Mb/s
  - 100BASE-TX
    - 802.3u-1995
    - up to 100m
    - Cat 5 / Cat 5e
  - 100BASE-T1
    - 802.3bw-2015
    - up to 15m
    - for automative, iot, etc.
  - there is also 100BASE‑FX
    - fiber optic cable
- gigabit ethernet, 1Gb/s
  - 1000BASE-T
    - 802.3ab-1999
    - up to 100m
    - Cat 5
  - 1000BASE-T1
    - 802.3bp-2016
    - up to 40m
    - Cat 6a
  - there are also fiber optic cable variants
- 2.5 gigabit and 5 gigabit ethernet, 2.5Gb/s 5Gb/s
  - 2.5GBASE-T and 5GBASE-T
    - 802.3bz-2016
    - up to 100m
    - Cat 5e and 6
- 10 gigabit ethernet, 10Gb/s
  - 10GBASE
    - 802.3ae-2002
    - mainly found in network switches
- 100 gigabit ethernet, 10Gb/s
  - 100GBASE
    - 802.3ba-2010
    - mainly found in network switches
- cables
  - Cat5e
    - 100mhz
    - up to 1Gb/s
  - Cat6
    - 250mhz
    - up to 1Gb/s
  - Cat6A
    - 500mhz
    - up to 10Gb/s
  - Cat7
    - 600mhz
    - up to 10Gb/s
  - Cat8
    - 2000mhz
    - up to 40Gb/s

## Gigabit

- cable should be at least cat 5e
- `ethtool <dev>`
  - `Supported link modes` and `Advertised link modes`
    - should have `1000baseT/Full`
  - `Link partner advertised link modes`
    - should have `1000baseT/Full`
  - `Speed: 1000Mb/s`
  - `Duplex: Full`
- or, `cat /sys/class/net/<dev>/speed`

## Link

     PHY <------MII------> MAC
physical layer         data link layer


PHY: sending/receiving frames over wires
MAC: decide whether the frame should be dropped or passed upper
