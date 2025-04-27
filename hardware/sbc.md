SBC
===

## Popular SoCs

- <https://hackerboards.com/>
- AllWinner
  - A64, 2015, A53 x4
  - H5,  2016, A53 x4
  - H7,  2017, A53 x4
- Amlogic
  - S905X, 2015, A53 x4
  - S905X3, 2019, A55 x4
- Broadcom
  - BCM2711, 20xx, A72 x4
  - BCM2712, 20xx, A76 x4
- Intel
  - Celeron N4500, 2021, Tremont x2
  - N100, 2023, Gracemont x4
- MediaTek
  - MT7986 (Filogic 830), 20xx, A53 x4
  - MT7988A (Filogic 880), 20xx, A73 x4
- NXP
  - i.MX 8M, 2017, A53 x4
  - i.MX 8 QuadMax, 2017, A72 x2 + A53 x4
- Rockchip
  - RK3328, 2017, A53 x4
  - RK3399, 2016, A72 x2 + A53 x4
  - RK3566, 2021, A55 x4
  - RK3568, 2021, A55 x4
  - RK3588, 2022, A76 x4 + A55 x4
- <https://www.cpubenchmark.net/> single thread results
  - 14900K: 4791
  - N100: 1966
  - N4500: 1394
  - A76 @ 2.4GHz: 1047
  - A72 @ 2.4GHz: 830
  - A55 @ 1.8GHz: 286
  - A53 @ 1.8GHz: 236

## Popular SBCs

- FriendlyElec NanoPi
  - R2S: RK3328
  - R4S: RK3399
  - R5S: RK3568B2
  - R6S: RK3588S
- Raspberry Pi
  - 4: BCM2711
  - 5: BCM2712
- Raxda Rock
  - 5B: RK3588
- Sinovoip Banana Pi
  - BPI-R3: MT7986
  - BPI-R4: MT7988A
  - BPI-M7: RK3588
- Xunlong Orange Pi
  - 3: RK3566
  - 5: RK3588
- Hardkernel ODROID
 - M1S: RK3566

## Mini Rack

- <https://mini-rack.jeffgeerling.com/>
- standard rack size
  - width: 19" (outer), 18.312" (screw hole-to-hole), 17.75" (inner)
  - 1U height: 1.75" (3 screw holes)
- mini rack size:
  - width: 10" (outer), 9.312" (screw hole-to-hole), 8.75" (inner)
  - 1U height: 1.75" (3 screw holes)
- devices
  - modem: 13.5x13.5x4.5
    - power: 12V DC
    - wan: coax, ethernet
  - router: 6.5x6.5x3
    - power: 5V3A USBC
    - wan: ethernet
    - lan: ethernet
  - switch: 10.5x10x3
    - power (back): 53.5V1.3A DC, 11x6x4
    - lan (front): ethernet x4
  - server: 9.5x7x3
    - power (side): 5V4A USBC
    - lan (front): ethernet
    - usb (front): USBA x2
- allocation
  - 1U: PDU with 3 outlets, USBC adapter
