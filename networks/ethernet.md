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
    - up to 2.5Gb/s
  - Cat6
    - 250mhz
    - up to 10Gb/s at 37 to 55 meters
  - Cat6A
    - 500mhz
    - up to 10Gb/s at 100 meters
      - thicker and harder to run
      - handle heat dissipation better for PoE++
  - Cat7
    - 600mhz
    - up to 10Gb/s
  - Cat8
    - 2000mhz
    - up to 40Gb/s
  - PoE
    - PoE (802.3af): 12.95W
    - PoE+ (802.3at): 25.50W
    - PoE++ (802.bt, 4PPoE): 51W
  - other considerations
    - inside the cable jacket, there are 4 twisted pairs of copper wires
    - shielded means foil inside the jacket to reduce EMI
      - stiffer and more expensive
      - usually not needed unless the run is near a power line or high-power
        device such as air conditioner
    - solid means the copper wires are solid rather than stranded to reduce
      noise
      - stiffer and more expensive
      - must have except for very short runs
    - plenum means the cables are rated for plenum air spaces required by fire
      code
      - fire-retardant and lower toxic fume
      - if not plenum, at least CMR-rated

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

## Home Network

- at each node,
  - performance
    - for modem, it should be able to handle ISP bandwidth
    - for ethernet, each node should be able to handle 1g/2.5g/10g
  - power consumption
    - passive/active cooling
    - electricity bill
- modem
- router
  - pfsense netgate advertises L3 forwarding, firewall, and vpn numbers
- switch
  - managed (for vlan, 802.1x, etc.) or not
  - number of ports and their speeds
  - unlike hub, a switch inspects the mac addrress and forwards packets to the
    right ports
- ap
- cabling
  - moca adapters for coax
  - Cat6 is good enough
    - Cat6A is more future proof but harder to run

## Link

     PHY <------MII------> MAC
physical layer         data link layer


PHY: sending/receiving frames over wires
MAC: decide whether the frame should be dropped or passed upper
