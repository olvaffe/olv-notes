Intel CPUs
==========

## CPU uArchs

- Big Cores
  - Sandy Bridge  - 2010 - 32nm
  - Ivy Bridge    - 2011 - 22nm
  - Haswell       - 2013 - 22nm
  - Broadwell     - 2014 - 14nm
  - Skylake       - 2015 - 14nm
  - Palm Cove     - 2018 - 10nm (skipped)
  - Sunny Cove    - 2019 - 10nm
  - Willow Cove   - 2020 - 10nm
  - Golden Cove   - 2021 - Intel 7 (was 10nm, but roughly TSMC's 7nm)
  - Raptor Cove   - 2022 - Intel 7
  - Redwood Cove  - 2023 - Intel 4
  - Lion Cove     - 2024 - Intel 20A?
  - Panther Cove  - 2025 - Intel 18A?
- Little Cores
  - Silvermont    - 2013 - 22nm
  - Airmont       - 2014 - 14nm
  - Goldmont      - 2016 - 14nm
  - Goldmont Plus - 2017 - 14nm
  - Tremont       - 2020 - 10nm
  - Gracemont     - 2021 - Intel 7
  - Crestmont     - 2023 - Intel 4
  - Skymont       - 2024 - Intel 20A?
  - Darkmont      - 2025 - Intel 18A?

## CPUs

- 11Q4~: Core 2xxx
  - Sandy Bridge, Gen6 GPU
- 12Q4~: Core 3xxx
  - Ivy Bridge, Gen7 GPU
- 13Q4~: Core 4xxx
  - Haswell, Gen7.5 GPU
  - Atom: Bay Trail (Valleyview), 2-10W, Silvermont, Gen7 GPU
- 14Q4~: Core 5xxx
  - Broadwell, Gen8 GPU
  - Atom: Braswell (Cherry Trail, Cherryview), 2-6W, Airmont, Gen8 GPU
- 15Q4~: Core 6xxx
  - Skylake, Gen9 GPU
- 16Q4~: Core 7xxx
  - Kaby Lake, Skylake uarch, Gen9.5 GPU
    - cros: poppy
  - Atom: Apollo Lake, 6-10W, Goldmont, Gen9.5 GPU
    - cros: reef, coral
- 17Q4~: Core 8xxx
  - Coffee Lake, Skylake uarch, Gen9.5 GPU
  - Whiskey Lake, Skylake uarch, Gen9.5 GPU
  - Kaby Lake R, Skylake uarch, Gen9.5 GPU
    - cros: fizz, rammus
  - Amber Lake Y, Skylake uarch, Gen9.5 GPU
  - Atom: Gemini Lake, Goldmont Plus, Gen9.5 GPU
    - cros: octopus
- 18Q4~: Core 9xxx
  - Coffee Lake Refresh, Skylake uarch, Gen9.5 GPU
  - Cannon Lake (skipped), Palm Cove, Gen10 GPU
- 19Q4~: Core 10xxx
  - Ice Lake, Sunny Cove, Gen11 GPU
  - Comet Lake, Skylake uarch, Gen9.5 GPU
    - cros: hatch, puff, drallion
  - Amber Lake Y, Skylake uarch, Gen9.5 GPU
- 20Q4~: Core 11xxx
  - Tiger Lake, Willow Cove, Gen12 GPU
    - cros: volteer
  - Rocket Lake, Sunny Cove, Gen12 GPU
  - Atom: Jasper Lake, 6-15W, Tremont, Gen11 GPU
    - cros: dedede
- 21Q4~: Core 12xxx
  - Alder Lake, Golden Cove, Gracemont, Gen12 GPU
    - cros: brya
- 22Q4~: Core 13xxx
  - Raptor Lake, Raptor Cove, Gracemont, Gen12 GPU
    - cros: skolas
  - Atom: Alder Lake N, 6-15W, Gracemont, Gen12 GPU
    - cros: nissa
- 23Q4~: Core 14xxx
  - Raptor Lake-R, Raptor Cove, Gracemont, Gen12 GPU
- 23Q4~: Core Ultra 1xx
  - Meteor Lake ,Redwood Cove, Crestmont, Gen12.5 GPU
    - cros: rex
- 24Q4~: Core Ultra 2xx?
  - Arrow Lake
- 25Q4~: Core Ultra 3xx?
  - Lunar Lake
  - Panther Lake
  - Nova Lake

## Product Lines

- Intel Core Numbering System
  - e.g., `Intel Core i7-13700K`
  - Brand
    - `Intel Core`
  - Brand Modifier
    - `i7`
  - Generation Indicator
    - `13`
  - SKU
    - `700`
    - in general, the larger the more features within the same generator
  - Product Line Suffix
    - `K`
- Since 12th gen, there are P-cores and E-cores
- Desktop Suffices
  - TDP 35W-150W
  - K, High performance, unlocked
  - F, Requires discrete graphics
  - S, Special edition
  - T, Power-optimized lifestyle
  - X/XE, Highest performance, unlocked
- Mobile Suffices
  - HX, Highest performance, all SKUs unlocked
  - HK, High performance, unlocked
  - H, High performance
    - TDP >=45W
  - P, Performance for thin & light
    - TDP <=28W
  - U, Power efficient
    - TDP <=15W
  - Y, Extremely low-power efficient
  - G1-G7, Graphics level (processors with newer integrated graphics technology)
- Embedded Suffices
  - E, Embedded
  - UE, Power efficient
  - HE, High performance
  - UL, Power efficient, in LGA package
  - HL, High performance, in LGA package
- other product line suffices
  - U (UP3)
    - ultra-thin laptop
    - TDP <=15W
  - U (UP4)
    - fanless
    - TDP >=9W
  - N
    - ultra-mobile
    - TDP >=6W

## Intel SoC design

- Long time ago, there were CPU, northbridge, and southbridge
  - northbrdige is connected to CPU via FSB and has
    - memory controller
    - PCIe controller (or AGP)
    - CPU frequency is FSB frequency times multiplier
  - southbridge is connected to northbridge via DMI and has
    - pci controller
    - lpc controller (replaces ISA)
    - spi controller (for BIOS flash)
    - smbus (for thermal sensors and fans)
    - dma controller (for pci and lpc devices)
    - i/o apic (for device interrupts)
    - SATA controller
    - RTC
    - HDA controller
    - USB controller
    - super io (keyboard, mouse, serial ports)
- Then there were CPU and PCH
  - northbridge is integrated into CPU die and is called uncore or system
    agent
  - southbridge is replaced by PCH
  - CPU uncore is connected to CPU core via QPI and has
    - LLC
    - memory controller
    - snoop agent
    - thunderbolt controller
  - PCH is connected to CPU via DMI and has
    - everything that southbridge has
- there is only CPU now
  - system-in-package design
  - CPU die contains CPU core and system agent
  - PCH is integrated into the CPU package as a PCH die
  - PCH die is connected to the CPU die via OPI
  - FWIW, wafers are "diced" into many "die"
- Chiplets
  - on Meteor Lake, there are 4 chiplets (called tiles)
    - SOC tile
    - CPU/Compute tile
    - GPU tile
    - IOE (IO expander) tile
  - interconnect between tiles
    - traditionally, tiles are side-by-side with interconnects on the same
      surface (2D)
    - in EMIB, tiles are side-by-side with interconnects below the tiles
      (2.5D stacking)
    - in Faveros, tiles are face-to-face (3D stacking)

## PCI devices

- CPU die
  - PCI host bridge (connecting system bus and PCI bus)
  - Gen GPU
  - HDA controller (for HDMI)
  - processor thermal subsystem
- PCH die
  - USB xHCI controller
  - USB EHCI controller
  - MEI controller

## Boot Process

- on power up,
  - PMC starts execution; the rest is held in reset state
    - Power Management Controller
  - then CSME starts execution
    - Converged Security and Management Engine
    - it is an i468 and it executes the CSME firmware on the ROM
    - root-of-trust
  - then CPU starts execution

## cpuid

- <https://en.wikipedia.org/wiki/CPUID>
- `cpuid` instruction implicitly uses `eax` and optionally `ecx`
  - `eax` specifies the leaf (category)
  - for some leaves, `ecx` specifies the subleaves
- `cpuid.h`
  - `for (unsigned int i = 0; i <= __get_cpuid_max(0, NULL); i++)`
  - `  if (__get_cpuid(i, ...)) ...;`
- `eax=0x0`: Highest Function Parameter and Manufacturer ID
  - it returns the highest leaf in `eax`
  - it returns the manufacturer id in `ebx`, `edx`, and `ecx`, 12 chars in
    total
    - `GenuineIntel` for intel
    - `AuthenticAMD` for amd
- `eax=0x1`: Processor Info and Feature Bits
  - `eax` holds the cpu signature
    - bit 0..3: stepping
    - bit 4..7: model
    - bit 8..11: family
    - bit 12..13: processor type
    - bit 16..19: extended model id
      - the higher bits of model
    - bit 20..27: extended family id
      - the higher bits of family
    - the same info is availabe in `/proc/cpuinfo`
    - microcode file naming: `<family>-<model>-<stepping>`
  - `ebx` holds the additional info
    - bit 0..7: band index
    - bit 8..15: clflush line size in qwords
    - more
  - `edx` and `ecx` hold the feature flags
    - fpe, tsc, msr, mtrr, pat, sse, sse2, sse3, sse4.1, sse4.2, x2apic,
      popcnt, aes-ni, avx, rdrnd, etc.
- `eax=0x2`: Cache and TLB Descriptor information
- `eax=0x3`: Processor Serial Number
- `eax=0x4` and `eax=b`: Intel thread/core and cache topology
- `eax=0x6`: Thermal and power management
- `eax=0x7`: Extended Features
  - fsgsbase, avx2, clflushopt, etc.
- `eax=0xd`: XSAVE features and state-components
- `eax=0x12`: SGX capabilities
- `eax=0x14`: Processor Trace
- `eax=0x19`: AES Key Locker features
- `eax=0x24`: AVX10 Features
