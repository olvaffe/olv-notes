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
- Little Cores
  - Silvermont    - 2013 - 22nm
  - Airmont       - 2014 - 14nm
  - Goldmont      - 2016 - 14nm
  - Goldmont Plus - 2017 - 14nm
  - Tremont       - 2020 - 10nm
  - Gracemont     - 2021 - Intel 7

## Product Lines

- Desktop
  - TDP 35W-150W
- Mobile
  - H
    - high-performance mobile workstations
    - TDP >=45W
  - P
    - perfomrance laptop
    - TDP <=28W
  - U (UP3)
    - ultra-thin laptop
    - TDP <=15W
  - U (UP4)
    - fanless
    - TDP >=9W
  - N
    - ultra-mobile
    - TDP >=6W

## Platforms

- Hybrid Platforms
  - Alder Lake
    - 2021
    - Golden Cove / Gracemont uArch
    - Gen12.5 GPU
  - Raptor Lake
  - Meteor Lake
  - Arrow Lake
  - Lunar Lake
- Big Platforms
  - Sandy Bridge
    - Gen6 GPU
  - Ivy Bridge
    - Gen7 GPU
  - Haswell
    - Gen7.5 GPU
  - Broadwell
    - Gen8 GPU
  - Skylake
    - Gen9 GPU
  - Kaby Lake
    - 2016
    - Skylake uArch
    - Gen9.5 GPU
    - Kaby Lake R (2017 refresh)
    - Amber Lake (2018 refresh)
  - Coffee Lake
    - 2017
    - Skylake uArch
    - Gen9.5 GPU
    - Whiskey Lake (2018 refresh?)
  - Cannon Lake (skipped)
    - 2018
    - Palm Cove
    - Gen10 GPU
  - Comet Lake
    - 2019
    - Skylake uArch
    - Gen9.5 GPU
  - Ice Lake
    - 2019
    - Sunny Cove uArch
    - Gen11 GPU
    - this is for mobile
      - there is Rocket Lake for desktops
  - Tiger Lake
    - 2020
    - Willow Cove uArch
    - Gen12 GPU
    - this is also for mobile
      - there is Rocket Lake for desktops
- Little Platforms
  - Bay Trail (Valleyview)
    - 2013
    - Silvermont uArch
  - Braswell (Cherry Trail, Cherryview)
    - 2014
    - Airmont uArch
  - Apollo Lake
    - 2016
    - Goldmont uArch
  - Gemini Lake
    - 2017
    - Goldmont Plus uArch
  - Jasper Lake
    - 2020
    - Tremont uArch

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
- there are still CPU and PCH for desktop; but there is only CPU for mobile
  - system-in-package design
  - CPU die contains CPU core and system agent
  - PCH is integrated into the CPU package as a PCH die
  - PCH die is connected to the CPU die via OPI

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
