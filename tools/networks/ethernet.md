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
    - for automative, iot, etc.
  - there are also fiber optic cable variants
- 2.5 gigabit and 5 gigabit ethernet, 2.5Gb/s 5Gb/s
  - 2.5GBASE-T and 5GBASE-T
    - 802.3bz-2016
    - up to 100m
    - Cat 5e and 6
  - there are other variants
- 10 gigabit ethernet, 10Gb/s
  - 10GBASE-T
    - 802.3an-2006
    - up to 100m
    - Cat 6a
  - there are other variants
- 25 gigabit, 40 gigabit and 100 gigabit ethernet, 40Gb/s, 100Gb/s
  - 25GBASE-T and 40GBASE-T
    - 802.3bq-2016
    - up to 30m
    - Cat 8
  - 100GBASE
    - 802.3ba-2010
    - mainly found in network switches
- 200, 400, 800 gigabit ethernet
  - data centers

## Twisted-Pair Cables

- twisted-pair cables
  - inside the cable jacket, there are 4 twisted pairs of copper wires
    - each pair is twisted to reduce EMI
  - solid means the copper wires are solid rather than stranded to reduce
    noise
    - stiffer and more expensive
    - must have except for very short runs
    - there is also CCA which is also bad
  - plenum (CMP, as opposed to CMR) means the cables are rated for plenum air
    spaces required by fire code
    - fire-retardant and lower toxic fume
    - if not CMP, at least CMR-rated
  - shielding reduces EMI and helps heat dissipation
    - but is stiffer and more expensive
    - usually not needed unless PoE+, or the run is near a power line or
      high-power device such as air conditioner
    - U/UTP: cable unshielded, each pair unshielded
    - F/FTP: cable foiled, each pair foiled
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
- Cat8
  - 2000mhz
  - up to 40Gb/s
- PoE
  - PoE (802.3af): 12.95W
  - PoE+ (802.3at): 25.50W
  - PoE++ (802.3bt, 4PPoE): 51W

## VLAN

- 802.11q
- an untagged/access port is a port where the incoming and outgoing frames
  are untagged
  - that is, the device connected to the port sends/receives untagged frames
  - the port is configured with a specific vlan id
    - for each incoming frame received from the connected device, the switch
      adds a tag, checks the destination, and forwards the frame to the
      right port
    - for each outgoing frame forwarded from another port, the switch checks
      the tag, strips the tag, and sends the frame
      - if the frame is untagged or has a wrong tag, the frame is dropped
        instead
- a tagged/trunk port is a port where the incoming and outgoing frames
  are tagged (except for native vlan)
  - that is, the device connected to the port sends/receives tagged frames
    (e.g., another switch's tagged port) except for native vlan
  - the port is configured with multiple vlan ids
    - for each incoming frame received from the connected device, the switch
      checks the tag, checks the destination, and forwards the frame to the
      right port
    - for each outgoing frame forwarded from another port, the switch checks
      the tag, and sends the frame
  - the port is also configured with a native vlan id
    - it works the same way as an untagged port
- no first-hand experience, but it feels like each port has a native vlan id
  and a list of allowed vlan ids
  - if a frame is not tagged, it is assumed to have the native vlan id

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
