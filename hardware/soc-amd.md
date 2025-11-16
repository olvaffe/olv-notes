AMD CPUs
========

## CPU uArchs

- Bulldozer   - 2011 - 32nm
- Piledriver  - 2012 - 32nm
- Steamroller - 2014 - 28nm
- Excavator   - 2015 - 28nm
- Zen         - 2017 - 14nm
- Zen+        - 2018 - 12nm
- Zen 2       - 2019 - 7nm
- Zen 3       - 2020 - 7nm
- Zen 3+      - 2022 - 6nm (apu only)
- Zen 4       - 2022 - 5nm
- Zen 5       - 2024 - 3nm

## CPUs and APUs

- 2016: A 9000 series
  - Bristol Ridge, 35-65W, Excavator+, GCN3
  - Stoney Ridge, 6-15W,, Excavator+, GCN3
    - cros: grunt
- 2017: Ryzen 1000 series
  - Whitehaven, 180W, Zen, no GPU
  - Summit Ridge, 65-95W, Zen, no GPU
- 2018: Ryzen 2000 series
  - Colfax, 180-250W, Zen+, no GPU
  - Pinnacle Ridge, 45-105W, Zen+, no GPU
  - Raven Ridge, 35-65W, Zen, GCN5
- 2019: Ryzen 3000 series
  - Castle Peak, 280W, Zen 2, no GPU
  - Matisse, 65-105W, Zen 2, no GPU
  - Picasso, 15-65W, Zen+, GCN5
    - cros: zork
  - Dali/Pollock (2020), 6-15W, Zen, GCN5
    - cros: zork
- 2020: Ryzen 4000 series
  - Renoir, 15-65W, Zen 2, GCN5
- 2021: Ryzen 5000 series
  - Chagall, 280W, Zen 3, no GPU
  - Vermeer, 65-105W, Zen 3, no GPU
  - Cezanne, 35-65W, Zen 3, GCN5
  - Cezanne-U, 10-25W, Zen 3, GCN5
  - Lucienne, 10-25W, Zen 2, GCN5
  - Barcelo (2022), 15W, Zen 3, GCN5
    - cros: guybrush
- 2022: Ryzen 6000 series
  - Rembrandt, 15-45W, Zen 3+, RDNA2
- 2023: Ryzen 7000 series
  - Storm Peak, 350W, Zen 4, no GPU
  - Raphael, 65-170W, Zen 4, RDNA2-display
    - the gpu has only 2 CUs and is mostly for display
  - Dragon Range, 45-75W, Zen 4, RDNA2-display
    - the gpu has only 2 CUs and is mostly for display
    - this is used on high-end mobile with discrete gpu
  - Phoenix, 15-54W, Zen 4, RDNA3
    - cros: myst
  - Mendocino, 15W, Zen 2, RDNA2
    - cros: skyrim
  - Rembrandt-R, 15-54W, Zen 3+, RDNA2
  - Barcelo-R, 15W, Zen 3, GCN5
- 2024: Ryzen 8000 series
  - Phoenix, 65W, Zen 4, RDNA3
  - Hawk Point, 15-54W, Zen 4, RDNA3
- 2024: Ryzen 9000 series and AI 300 series
  - Granite Ridge, 65-170W, Zen 5, RDNA2-display
  - Strix Point, 28W, Zen 5, RDNA3.5

## Chromebooks

- grunt
  - A6-9220C
  - 2C2T
  - gpu `1002:98e4`
  - `CHIP_STONEY|AMD_IS_APU`
  - `AMDGPU_FAMILY_CZ`
- zork
  - Ryzen 7 3700C
  - 4C8T
  - gpu `1002:15d8`
  - `CHIP_RAVEN|AMD_IS_APU`
  - `AMDGPU_FAMILY_RV`
- guybrush
  - Ryzen 5 5625C
  - 6C12T
  - gpu `1002:15e7`
  - `CHIP_RENOIR|AMD_IS_APU`
  - `AMDGPU_FAMILY_RV`
- skyrim
  - Ryzen 5 7520C
  - 4C8T
  - gpu `1002:1506`
  - `CHIP_ID_DISCOVERY|AMD_IS_APU`
  - `AMDGPU_FAMILY_GC_10_3_6`
  - gpu info
    - `num_shader_engines = 1`
    - `num_shader_arrays_per_engine = 1`
    - `num_cu_per_sh = 2`
    - `cu_active_number = 2`
    - each cu has 2 SIMD32, meaning it can perform `2*32*2=128` ops per cycle
    - at 1900MHZ, it peaks at 243 GOPS
- typical hw
  - intel: N4000 (2C@1.1G), N4120 (4C@1.1G), N4500 (2C@1.1G), N6000 (4C@1.1G)
  - amd: 9120C (2C@1.6G), 3015Ce (2C@1.2G)
  - mediatek: MT8173 (2C+2C), MT8183 (4C+4C)
  - memory: 4GB
  - storage: 32GB, followed by 16GB/64GB

## Product Lines

- 2023 Mobile Numbering System
  - 7640U
  - first number is model year
    - 7 is 2023
    - 8 is 2024
  - second number is market segment
    - 1 is athlon silver
    - 2 is athlon gold
    - 3/4 are ryzen 3
    - 5/6 are ryzen 5
    - 7 is ryzen 7
    - 8 is ryzen 7/9
    - 9 is ryzen 9
  - third number is architecture
    - 1 is zen 1/1+
    - 2 is zen 2
    - 3 is zen 3/3+
    - 4 is zen 4
    - 5 is zen 5
  - fourth number is feature isolation
    - 0 is lower end within the segment
    - 5 is higher end within the segment
  - the letter is form factor / TDP
    - HX is 55W+ (extreme gaming/creator)
    - HS is 35W-45W (gaming/creator)
    - U is 18W-28W (thin and light)
    - C is 18W-28W (chromebook)
    - e is 9W (fanless variant of U)
- Ryzen 7 PRO 5850U 
  - product family (Ryzen 7)
    - 3 entry level
    - 5 mid-range
    - 7 high end
    - 9 ehthusiast
  - modifier (PRO)
    - PRO business
    - (none) consumer
  - generation (5)
  - perfomrance level (8)
  - model (50)
  - product line (U)
    - U ultra-thin mobile (TDP 15W)
    - H high-performance slim mobile (TDP 35W)
    - H high-performance mobile (TDP 45W)
    - GE energy-efficient deskop (TDP 35W)
    - G deskop (TDP 65W)

## SMU, System Management Unit

- SMU consists of MP0, MP1, and MP2
- MP0 is known as PSP, platform security processor
  - aka ASP, AMD secure processor
  - appears to be a cortex-a5
- MP1 is often referred to simply as SMU
  - appears to be a xtensa cpu
  - responsible for power, clock, thermal, etc.
- MP2 is known as SFH, sensor fusion hub
  - responsible for external sensors

## Zen 4 Overclocking

- <https://skatterbencher.com/2022/09/26/raphael-overclocking-whats-new/>
- clocking topo: `CGPLL` converts external crystal 48mhz to
  - `CCLK -> VCO -> CCX -> cpu cores`
  - `PCIECLK -> pcie`
  - `FCLK -> infinity fabric`
  - `UMCCLK -> UCLK -> memory controller`
  - `UMCCLK -> MCLK -> dram`
  - `GFXCLK -> igpu`
  - `USB -> usb`
- voltage topo: there are 4 power rails from vrm
  - `VDDCR` is converted to `VDDCR_CPU` (cores) and `VDDCR_VDDM` (caches)
  - `VDDCR_SOC` is converted to `VDDCR_SOC` (umc, smu, etc.) and `VDDCR_GFX` (igpu)
  - `VDDCR_MISC` is converted to `VDDGs` (one for each infinity fabric interconnect)
  - `VDDIO_MEM_S3` is converted to `VDDP_DDR` (dram)
- AMD Serial VID Interface 3 (SVI3)
  - an interface (similar to i2c) to control soc voltage regulators
- Precision Boost
  - the task is performed by SMUs in each die
  - Sustained Power Limit (SPL) is tdp
  - Package Power Tracking (PPT) is the max power draw for boost, with
    limiting factor being thermal
  - Electrical Design Current (EDC) is the peak current, with limiting factor
    being vrm
  - Thermal Design Current (TDC) is the sustained current, with limiting
    factor being vrm and thermal

## Ryzen 5 9600X PCI and USB

- lspci and lsusb
  - 00:00.2 IOMMU: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge IOMMU
  - 00:01.2 PCI bridge: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge GPP Bridge
    - 01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller S4LV008[Pascal]
  - 00:02.1 PCI bridge: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge GPP Bridge
    - 02:00.0 PCI bridge: Advanced Micro Devices, Inc. [AMD] 600 Series Chipset PCIe Switch Upstream Port (rev 01)
      - 07:00.0 Ethernet controller: Intel Corporation Ethernet Controller I225-V (rev 03)
      - 08:00.0 Network controller: MEDIATEK Corp. MT7922 802.11ax PCI Express Wireless Network Adapter
      - 09:00.0 USB controller: Advanced Micro Devices, Inc. [AMD] 600 Series Chipset USB 3.2 Controller (rev 01)
        - Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub (12 ports)
          - Bus 001 Device 002: ID 0b05:1a5c ASUSTek Computer, Inc. USB Audio
          - Bus 001 Device 003: ID 0b05:19af ASUSTek Computer, Inc. AURA LED Controller
          - Bus 001 Device 004: ID 0489:e0e2 Foxconn / Hon Hai Wireless_Device
        - Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub (5 ports)
      - 0a:00.0 SATA controller: Advanced Micro Devices, Inc. [AMD] 600 Series Chipset SATA Controller (rev 01)
  - 00:08.1 PCI bridge: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge Internal GPP Bridge to Bus [C:A]
    - 0b:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Granite Ridge [Radeon Graphics] (rev c6)
    - 0b:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Radeon High Definition Audio Controller [Rembrandt/Strix]
    - 0b:00.2 Encryption controller: Advanced Micro Devices, Inc. [AMD] Family 19h PSP/CCP
    - 0b:00.3 USB controller: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge USB 3.1 xHCI
      - Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub (2 ports)
      - Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub (2 ports)
    - 0b:00.4 USB controller: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge USB 3.1 xHCI
      - Bus 005 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub (2 ports)
      - Bus 006 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub (2 ports)
  - 00:08.3 PCI bridge: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge Internal GPP Bridge to Bus [C:A]
    - 0c:00.0 USB controller: Advanced Micro Devices, Inc. [AMD] Raphael/Granite Ridge USB 2.0 xHCI
      - Bus 007 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub (1 port)
  - 00:14.0 SMBus: Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller (rev 71)
  - 00:14.3 ISA bridge: Advanced Micro Devices, Inc. [AMD] FCH LPC Bridge (rev 51)
- <https://www.amd.com/en/products/processors/desktops/ryzen/9000-series/amd-ryzen-5-9600x.html>
  - two `USB 3.1 xHCI` each providing 2 ports usb3 gen2 (10G)
  - one `USB 2.0 xHCI` providing 1 port
- <https://www.amd.com/en/products/processors/chipsets/am5.html#specs>
  - one `USB 3.2 Controller` providing
    - 5-port usb3 gen2x2 (20G)
    - 12-port usb2
  - one `SATA Controller`
- <https://rog.asus.com/us/motherboards/rog-strix/rog-strix-b650e-i-gaming-wifi-model/>
  - one `USB Audio`
  - one `AURA LED Controller`
  - one `Foxconn / Hon Hai Wireless_Device` (mtk)
  - one `Ethernet controller` (intel)
  - one `Network controller` (mtk)
  - internal usb headers providing one typec (10G), two usb3 (5G), two usb2
    - all connected to `USB 3.2 Controller`
  - one usb3 and one typec in back io far right
    - both connected to `USB 3.2 Controller`
  - two usb2 in back io far left
    - top one connected to `USB 3.2 Controller`
    - bottom one connected to `USB 2.0 xHCI`
  - three usb3 and one typec in back io center
    - middle two usb3 connected to `USB 3.1 xHCI`
    - the other usb3 and typec connected to another `USB 3.1 xHCI`
