PC DIY
======

## CPUs

- Intel Core
  - i5: $200 ~ $300
  - i7: $300 ~ $500
- AMD Ryzen
  - 5: $100 ~ $200
  - 7: $200 ~ $500
- TDP
  - OEMs design the cooler to handle TDP watts
    - this is the sustainable watts that cpu can draw
  - cpu draws peak watts when all cores run at boost freq
    - this is not sustainable
  - when cpu draws peak watts, it keeps getting hotter due to the difference
    of `(peak - tdp)` watts
  - until cpu thermal-trottles itself
    - 100C on Core
    - 95C on Ryzen
  - when OEMs design the cooler to handle a specific TDP, high-end and low-end
    cpus having the same TDP have roughly the same sustainable performance
    - the big difference between high-end and low-end is before
      thermal-throttle

## CPU Coolers

- $30 ~ $80
- clearances
  - cases have cooler height clearances
    - small cases require low-profile coolers
  - coolers may have ram and io clearances
    - 90x90mm coolers are fine
    - 120x120mm coolers on itx have either ram clearnace or back panel io
      clearnace
- low-profile performance
  - 90x90mm is good enough to sustain TDP 65W cpus
    - height clearance: 37mm - 65mm
  - 120x120mm is needed for 65W cpus to stay at boost freq longer
    - height clearance: 65mm - 70mm
- for small builds emphasizing cpu performance,
  - pick 65W cpu
  - pick 120x120mm cooler
  - pick cases with high cooler height clearance
  - pick low-profile ram or mb with low-profile back io

## Motherboards

- LGA 1700
  - $100 ~ $300
- AM5
  - $100 ~ $300
- front panel headers
  - buttons / leds
  - audio jack
  - USB-As
  - USB-Cs
    - some motherboards miss this header

## Memories

- DDR5 32GB
  - $80 ~ $400
- DDR4 32GB
  - $40 ~ $200
- considerations
  - for ddr5, cpus support 5200MHz/5600MHz officially
    - no need to go beyond 6000MHz and CL 32
  - for ddr4, cpus support 3200MHz officially
    - no need to go beyond 3600MHz and CL 16
  - height can be limited by cases or cpu coolers

## Storage

- m2 1TB
  - $40 ~ $100
- m.2 standard
  - form factors
    - 2280, 22x80mm
  - interfaces
    - pcie + nvme
      - 3.0 x4: 4GB/s
      - 4.0 x4: 8GB/s
      - 5.0 x4: 16GB/s
    - sata + ahci
- m.2 ssd components
  - controller
  - dram as cache
  - nand for storage
    - SLC/TLC/QLC: affect speed and durability
    - TBW: terabytes written

## Cases

- micro-atx
  - $100 ~ $200
- mini-itx
  - $100 ~ $300
- <https://caseend.com/>
- sub-5L
  - deskmini x300: 1.92L
    - 46mm cpu cooler
    - igpu only
    - builtin psu
    - builtin mb
    - like a mini pc
  - Densium APU: 4L
    - 67mm cpu cooler
    - igpu only
    - pico/gan/flex psu
  - lzmod a24 v4: 4.3L
    - 66mm cpu cooler
    - lp gpu
    - flex psu
  - chopin max: 4.4L
    - 54mm cpu cooler
    - igpu only
    - builtin psu
  - cooj mq4: 4.9L
    - 69mm cpu cooler
    - igpu only
    - flex psu
  - velka 5: 4.9L
    - 37mm cpu cooler
    - dgpu in sandwich layout
    - flex psu
- sub-10L
  - cooj mq6: 6.8L
    - 72mm cpu cooler
    - dgpu in sandwich layout
    - flex psu
- sub-20L
  - lian li A4-H2O: 11L
    - aio cpu cooler
    - dqpu in sandwich layout
    - sfx/sfx-l psu
  - fractal ridge: 12.6L
    - 70mm cpu cooler
    - dgpu in dual-chamber
    - sfx/sfx-l psu
  - nr200p: 18.5L
- 20L+
  - <https://www.reddit.com/r/mffpc/>

## PSUs

- 750W
  - $100 ~ $180
- SFX
  - $80 ~ $200
- <https://cultists.network/140/psu-tier-list/>
- form factors
  - ATX: 150x86mm, depth is 140mm but can vary
  - SFX: about 125x64mm, depth is 100mm
  - TFX: about 85x65mm, depth is 175mm
  - flex: 81.5x40.5mm, depth is 150mm
  - 1U: less than 1.75in (44.45mm) in height
    - comparing to flex, usually wider, deeper, noisier, and more powerful
  - gan: 55x25mm, depth is 170mm
  - pico: requires external ac-dc adapter

## GPUs

- mainstream
  - $300 ~ $600

## Example

- CPU: Ryzen 5 5600G
  - $140
- CPU Cooler: Thermalright AXP120-X67
  - $50
- MB: GIGABYTE B550I AORUS PRO AX
  - $200
  - CPU
    - 2x DDR4 3200 DIMMs
    - 1x PCIe 3.0 x16
      - 5600G has a total of 24 PCIe 3.0 lanes
      - 16 lanes are usually for graphics
      - 4 lanes are usually for M.2
      - the rest 4 lanes are for the chipset
        - B550 has 10 PCIe 3.0 lanes as well by itself
        - but when communicating to the cpu, it has to use the 4 lanes
          provided by the cpu
    - 1x M.2 PCIe 3.0 x4
    - 1x DisplayPort 1.4
    - 2x HDMI 2.1
    - 4x USB 3.2 Gen1
  - AMD B550
    - 1x M.2 PCIe 3.0 x4
    - 4x SATA
    - 1x USB-C 3.2 Gen2
    - 1x USB-A 3.2 Gen2
    - 2x USB-A 3.2 Gen1 (front)
    - 2x USB-A 2.0 (front)
  - Realtek MT7921K / AMD RZ608
    - 802.11ax
    - BT 5.2
  - Realtek ALC1220-VB hda codec
  - Realtek RTL8125 nic
  - other internal connectors
    - 1x 24-pin main power
    - 1x 8-pin cpu power
    - 1x CPU fan header
    - 2x system fan headers
    - 2x LED headers
    - 1x front panel, audio, usb
- RAM: G.SKILL Ripjaws 32GB
  - $90
  - 2 x 16GB
  - DDR4 3600
  - CL16
- Storage: Samsung 980 PRO M.2 2280 2TB PCIe Gen 4.0 x4
  - $140
- Case: Cooj Sparrow-MQ4 4.9L
  - $169
  - 4.9L
- PSU: EnhanceENP-7025B 250W
  - $60
- WLAN Antenna:
  - $20
- Total
  - `$140+$50+$200+$90+$140+$169+$60+$20 = $869`
