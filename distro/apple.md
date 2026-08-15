# Apple

## Disk Layout

- Partition 1: iSC container
  - iSCPreboot: mounted to `/System/Volumes/iSCPreboot`
    - used only for recovery or fw update
  - xarts: mounted to `/System/Volumes/xarts`
    - cross-system artifacts
  - Hardware: mounted to `/System/Volumes/Hardware`
    - hw diagnostics, telemetry, and calibration data
- Partition 2: System Recovery
  - Recovery: not mounted
    - fallback recovery environment
- Partition 3: macOS container
  - Macintosh HD: mounted to `/`
    - read-only verified system volume
  - Data: mounted to `/System/Volumes/Data`
    - user data
  - Preboot: mounted to `/System/Volumes/Preboot`
    - iBoot2, kernel, dt
  - Recovery: mounted to `/System/Volumes/Recovery`
    - macos recovery environment
  - VM: mounted to `/System/Volumes/VM`
    - swapfiles

## Boot Sequence

- Boot ROM / SecureROM
  - read-only mask rom
  - cpu starts execution from boot rom on boot
  - it initializes soc, power, and sram
  - it loads and cryptographically verifies iBoot1 from spi nor
- iBoot1
  - it initializes dram and storage controller (apple fabric)
  - it loads and cryptographically verifies iBoot2 from Preboot volume
- iBoot2
  - it initializes display, battery, and input
  - it loads and cryptographically verifies kernel and dt from Preboot volume
- XNU/Darwin kernel
  - it runs at EL1

## 1TR (One True Recovery)

- recovery modes
  - normal recovery: boots to macos `Recovery` volume
  - 1TR recovery: boots to macos `Recovery` volume with physical presence 
  - fallback 1TR: boots to fallback `Recovery` volume
  - if storage is wiped, iBoot1 inits display and prompts to use DFU mode
  - if spi nor is wiped, Boot ROM enters DFU mode automatically

## Asahi

- installation
  - resize `macOS` partition
  - create `Stub` partition formatted as APFS
    - Preboot: iBoot2, m1n1
    - Recovery: regular macos recovery environment
  - create esp partition formatted as FAT
  - create linux partition formatted as BTRFS
  - it tells iBoot1 to find iBoot2 from `Stub`
- boot sequence
  - iBoot2 loads m1n1 instead of XNU kernel
  - iBoot2 jumps to m1n1 directly, keeping at EL2
  - m1n1 translates apple dt to linux dt, loads uboot from esp, jumps to it
