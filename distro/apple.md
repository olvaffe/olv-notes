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

## Keyboard

- from left to right, there are 4 mod keys: `fn`, `ctrl`, `alt`, and `cmd`
  - `fn` changes keycodes
    - function keys become `F1..F12`
    - arrow keys become pageup, pagedown, home, and end
  - `fn` talks to desktop shell
    - `fn-a` toggles dock
    - `fn-c` toggles control center
    - `fn-h` toggles desktop
    - `fn-n` toggles notification center
  - `ctrl` sends ascii control codes in terminal
    - `ctrl-a` is home
    - `ctrl-b` is left
    - `ctrl-c` is sigint
    - `ctrl-d` is eof
    - `ctrl-e` is end
    - `ctrl-f` is right
    - `ctrl-l` is clear
    - `ctrl-m` is enter
    - `ctrl-n` is down
    - `ctrl-p` is up
    - `ctrl-w` is delete-a-word
    - etc.
  - `alt` means alternatives
    - `alt-left` moves back a word in terminal (escape sequence)
    - `alt-right` moves forward a word in terminal (escape sequence)
    - `alt-brightness` opens display settings
    - `alt-volume` opens audio settings
  - `cmd` sends various gui commands
    - `cmd-a` is all
    - `cmd-c` is copy
    - `cmd-f` is find
    - `cmd-m` is minimize
    - `cmd-o` is open
    - `cmd-p` is print
    - `cmd-q` is quit
    - `cmd-s` is save
    - `cmd-t` is tab
    - `cmd-v` is paste
    - `cmd-w` is close
    - `cmd-z` is undo
    - `cmd-,` is settings
    - `cmd-space` is spotlight

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

- `/dev/disk0`: physical disk
  - `/dev/disk0s1`: 0.5G, `Apple_APFS_ISC`, not mounted
  - `/dev/disk0s2`: remaining, `Apple_APFS`, not mounted
  - `/dev/disk0s3`: 5.5G, `Apple_APFS_Recovery`, not mounted
- `/dev/disk1`: synthesized disk for `Apple_APFS_ISC`
  - `/dev/disk1s1`: 8M, `iSCPreboot`, `/System/Volumes/iSCPreboot`
    - iBoot1 boot polices, etc.
  - `/dev/disk1s2`: 8M, `xART`, `/System/Volumes/xarts`
    - used by SEP (secure enclave processor) to store extended anti-replay
      tables
  - `/dev/disk1s3`: 4M, `Hardware`, `/System/Volumes/Hardware`
    - hw diagnostics, telemetry, and calibration data
  - `/dev/disk1s4`: 32K, `Recovery`, not mounted
- `/dev/disk2`: synthesized disk for `Apple_APFS_Recovery`
  - `/dev/disk2s1`: 2G, `Recovery`, not mounted
    - fallback recoveryOS
  - `/dev/disk2s2`: 2M, `Update`, not mounted
    - staging area for `Recovery` update
- `/dev/disk3`: synthesized disk for `Apple_APFS`
  - `/dev/disk3s1`: remaining, `Macintosh HD - Data`, `/System/Volumes/Data`
    - everything mutable
  - `/dev/disk3s2`: 100M, `Update`, `/System/Volumes/Update`
    - staging area for `Macintosh HD` update
  - `/dev/disk3s3`: 16G, `Macintosh HD`, not mounted directly
    - `/dev/disk3s3s1`: same, snapshot, `/`
      - signed system volume, aka SSV
      - a verified, read-only snapshot of `/dev/disk3s3`
  - `/dev/disk3s4`: 16G, `Preboot`, `/System/Volumes/Preboot`
    - iBoot2, kernel, dt
  - `/dev/disk3s5`: 4G, `Recovery`, not mounted
    - primary recoveryOS, aka 1TR
  - `/dev/disk3s6`: remaining, `VM`, `/System/Volumes/VM`
    - swap

## Boot Sequence

- BootROM / SecureROM
  - read-only mask rom
  - cpu starts execution from boot rom on boot
  - it initializes soc, power, and sram
  - it loads and cryptographically verifies iBoot1 from spi nor
- iBoot1 / LLB
  - it initializes dram, storage controller (apple fabric), and SEP
  - it mounts `iSCPreboot` to find boot policy (for A/B boot)
  - it loads and cryptographically verifies iBoot2 from `Preboot` volume
- iBoot2
  - it initializes display, battery, and input
  - it loads and cryptographically verifies kernel, dt, and SSV root hash also
    from `Preboot` volume
- XNU/Darwin kernel
  - it runs at EL1
  - it mounts SSV (snapshot of `Macintosh HD` volume) to `/`

## System Update

- before reboot,
  - it downloads updates to `Update` volume
  - it updates fallback recoveryOS (`/dev/disk2s1`) first
  - it updates `Macintosh HD` (`/dev/disk3s3`) directly
    - remember that we mount a read-only snapshot of it and is unaffected
  - it creates another snapshot and root hash for the updated volume
  - it creates another boot folder in `Preboot` (`/dev/disk3s4`)
  - it updates the boot policy in `iSCPreboot` (`/dev/disk1s1`)
  - it updates primary recoveryOS to (`/dev/disk3s5`)
- upon reboot,
  - iBoot1 loads the new iBoot2
  - iBoot2 loads the new kernel
    - iBoot2 also checks the `Update` volume for fw update, which can flash
      new iBoot1 to spi nor
  - kernel mounts the new SSV

## Recovery

- recovery modes
  - normal recovery: boots to macos `Recovery` volume
  - 1TR recovery: boots to macos `Recovery` volume with physical presence
  - fallback 1TR: boots to fallback `Recovery` volume
  - if storage is wiped, iBoot1 inits display and prompts to use DFU mode
  - if spi nor is wiped, Boot ROM enters DFU mode automatically
- 1TR stands for One True Recovery

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
