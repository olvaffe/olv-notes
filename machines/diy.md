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
- Ryzen 7 7700
  - <https://www.amd.com/en/product/12751>
  - Arch: Zen 4, 5nm
  - Clock: 3.8GHz
  - TDP: 65W
  - RAM: DDR5-5200
  - GPU: RDNA2, 2CU, 2200MHz
  - Connectivity:
    - PCIe
      - 5.0: 28 lanes
        - 4 lanes are reserved for chipset, which also caps the bandwidth of
          the chipset
      - chipset provides more lanes
    - USB
      - 3.2 Gen2 (10Gbps): x4
      - 2.0 (480Mbps): x1
      - chipset provides more ports
- Ryzen 5 5600G
  - <https://www.amd.com/en/product/11176>
  - Arch: Zen 3, 7nm
  - Clock: 3.9GHz
  - TDP: 65W
  - RAM: DDR4-3200
  - GPU: GCN5, 7CU, 1900MHz
  - Connectivity:
    - PCIe
      - 4.0: 24 lanes
        - 4 lanes are reserved for chipset
      - chipset provides more lanes
    - USB
      - 3.2 Gen1 (5Gbps): x4
      - chipset provides more ports
- AMD Ryzen 9 6900HX
  - <https://www.amd.com/en/product/11541>
  - Arch: Zen 3+, 6nm
  - Clock: 3.3GHz
  - TDP: 45W
  - RAM: LPDDR5-6400
  - GPU: RDNA2, 12CU, 2400MHz
  - Connectivity:
    - PCIe
      - 4.0: 20 lanes
        - 4 lanes are reserved for chipset
      - chipset provides more lanes
    - USB
      - 3.2 Gen2 (10Gbps): x2
      - 2.0 (480Mbps): x4
      - chipset provides more ports

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
- decibels
  - office background
    - meditation: 30dB
    - desk: 40dB
    - kitchen: 55dB
  - x1 carbon gen 9 fan
    - 0rpm / background: 30dB
    - 4500rpm: 30dB (noticeable)
    - 5500rpm: 35dB (acceptable)
    - 6500rpm: 40dB (a bit annoying)
  - white noise
    - muted / background: 30dB
    - 35dB: noticeable
    - 40dB: acceptable
    - 45dB: a bit annoying
    - 50dB: annoying
  - it seems frequencies and background decibels both play a role
    - for fan noises in a 30dB room, 45dB from a distance is acceptable

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
    - SLC/MLC/TLC/QLC: 1/2/3/4-bit cells
      - SLC is fastest and the most durable, but costs more
    - TBW: terabytes written
- Samsung
  - 960 EVO vs PRO
    - controller: polaris, pcie 3.0 x4, nvme 1.2
    - dram: 1GB LPDDR3 / TB
    - nand: tlc
    - TBW: 400 vs 800
    - warranty: 3yr vs 5yr
  - 970 EVO vs PRO
    - controller: phoenix, pcie 3.0 x4, nvme 1.3
    - cache: 1GB LPDDR4 / TB
    - nand: tlc vs mlc
    - TBW: 600 vs 1200
    - warranty: 5 years
  - 980
    - controller: in-house, pcie 3.0 x4, nvme 1.4
    - cache: HMB (uses system ram)
    - nand: tlc
    - TBW: 600
    - warranty: 5 years
  - 980 PRO
    - controller: elpis, pcie 4.0 x4, nvme 1.3c
    - cache: 1GB LPDDR4 / TB
    - nand: v6 tlc
    - TBW: 600
    - warranty: 5 years
  - 990 PRO
    - controller: pascal, pcie 4.0 x4, nvme 2.0
    - cache: 1GB LPDDR4 / TB
    - nand: v7 tlc
    - TBW: 600
    - warranty: 5 years

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
- flex psu noise levels:
  - Enhance ENP-7520B: 57dB at 200W
  - Enhance ENP-7660B: 33dB at 400W

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
- RAM: CORSAIR Vengeance LPX 32GB
  - $90
  - 2 x 16GB
  - DDR4 3600 / CL16
- Storage: Samsung 990 PRO 1TB
  - $100
- Case: Cooj Sparrow-MQ4 4.9L
  - $169
  - 4.9L
- PSU: Enhance ENP-7660B 600W
  - $170
- WLAN Antenna:
  - $20
- Total
  - `$140+$50+$200+$90+$100+$169+$170+$20 = $939`
