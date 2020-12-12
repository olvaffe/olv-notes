Ethernet
========

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
