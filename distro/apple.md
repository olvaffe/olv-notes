# Apple

## Initial Setup

- System Settings
  - General
    - About -> Name
    - Software Update
    - Date & Time
    - Language & Region
    - Login Items & Extensions
    - Sharing -> Local hostname
  - Appearance
  - Desktop & Dock
  - Menu Bar
  - Wallpaper
  - Sound
  - Lock Screen
  - Keyboard
  - Mouse
  - Trackpad
- Homebrew
  - bash
    - `brew install bash`
    - `echo /opt/homebrew/bin/bash >> /etc/shells`
    - `chsh -s /opt/homebrew/bin/bash`
  - middleclick
    - `brew install --cask middleclick`
  - aerospace
    - `brew install --cask nikitabobko/tap/aerospace`
- manual apps
  - alacritty
    - <https://github.com/alacritty/alacritty/releases>

## CLI Tools

- system
  - `system_profiler` shows system profile
    - `-listDataTypes` lists collected datatypes
      - `SPSecureElementDataType` for apple pay
      - `SPApplicationsDataType` for apps
      - `SPAudioDataType` for audio
      - `SPBluetoothDataType` for bt
      - `SPCameraDataType` for camera
      - `SPCardReaderDataType` for sd
      - `SPEthernetDataType` for ether
      - `SPFirewallDataType` for firewall
      - `SPDisplaysDataType` for gpu
      - `SPHardwareDataType` for machine
      - `SPMemoryDataType` for ram
      - `SPNVMeDataType` for nvme
      - `SPNetworkDataType` for network
      - `SPPCIDataType` for pci
      - `SPPowerDataType` for power
      - `SPSPIDataType` for spi (keyboard)
      - `SPStorageDataType` for storage
      - `SPThunderboltDataType` for usb4
      - `SPUSBHostDataType` for usb3
- hw
  - `ioreg` for io registry
    - it shows the tree of objects without props by default
    - to list all usb devices, `ioreg -r -d1 -c IOUSBHostDevice`
    - `-d <depth>` limits the tree depth
    - `-c <class>` shows props of matching objects
      - `-r` shows matching objects as roots
        - `-t` shows the path leading to matching objects
  - `diskutil` for disk
- network
  - `scutil` for low-level system configuration
    - `--get`
      - `HostName` is hostname
      - `LocalHostName` is mdns name
      - `ComputerName` is pretty name
    - `--dns` shows DNS resolvers
  - `networksetup` for high-level network
    - `-listallhardwareports` lists hw devs
    - `-listnetworkserviceorder` lists sw services
    - `-getinfo <srv>` gets sw service info
  - `dig`, `ifconfig`, `netstat`, etc.

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

## Homebrew

- `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
  - it creates and chowns `/opt/homebrew`
  - it `xcode-select --install` and `xcode-select --switch`
  - it adds `/etc/paths.d/homebrew`
  - it git clones <https://github.com/Homebrew/brew> to `/opt/homebrew`
- post-install setup
  - `/opt/homebrew/bin/brew shellenv bash` shows the required envs
  - `brew analytics off`
- homebrew installs to `/opt/homebrew` mostly
  - `/opt/homebrew/Cellar/<formula>/<version>` are installed formulas
  - `/opt/homebrew/Caskroom/<cask>/<version>` are installed casks
    - the real `.app` is under `/Applications`
- terms
  - a formula is a package definition that builds from source
    - a bottle is a pre-built formula package
  - a cask is a package definition that repackages binaries
  - a tap is a third-party repo for formulas/casks
