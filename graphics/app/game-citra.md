Citra
=====

## Custom Firmware (CFW)

- step 1: running any homebrew app
  - <https://github.com/zoogie/super-skaterhax>
  - skater is the codename of the webkit-based browser
  - there is a UAF in webkit.  Through the UAF and ROP (return-oriented
    programming) chains, we can execute a custom code (`/arm11code.bin`) that
    loads and runs an homebrew app
    (`/browserhax_hblauncher_ropbin_payload.bin`)
  - the homebrew app is `hbmenu`, which can select and launch any homebrew app
    (`*.3dsx`) from under `/3ds`
- step 2: installing boot9strap
  - <https://github.com/zoogie/unSAFE_MODE>
  - the safe mode (recovery mode) has a bug in wifi slot proxy url parsing
    - `slotTool` is a homebrew that sets malformed proxy urls for wifi slots
  - once we have malformed proxy urls, we can trigger the exploit from safe
    mode to execute `/usm.bin`?
  - `/usm.bin` chainloads `SafeB9SInstaller.bin` which installs
    `boot9strap.firm` to the internal storage
- details
  - <https://github.com/SciresM/boot9strap>
    - the bootrom has the root-of-trust key
    - there is a bug in the bootrom that allows loading unsigned firmware
    - the exploit is called sighax, signature hack
    - `boot9strap.firm` is an implementation of sighax
    - `boot9strap.firm` loads `/boot.firm`
  - <https://github.com/LumaTeam/Luma3DS>
    - `Luma3DS` is installed as `/boot.firm`, to be loaded by `boot9strap`
    - it is a custom firmware that can chain load another custom firmware or
      chain load the stock firmware with live-patching
    - it can launch the homebrew app, `/boot.3dsx`, as the homebrew launcher
  - <https://github.com/devkitPro/3ds-hbmenu>
    - `hbmenu` is an homebrew app that launches other homebrew apps
    - it can launch any homebrew app under `3ds/`
  - <https://github.com/d0k3/GodMode9>
    - `GodMode9` is a custom firmware that provides a full-access file
      browser

## Dumping

- dumping a cartridge
  - <https://citra-emu.org/wiki/dumping-game-cartridges/>
  - `GodMode9` can decrypt `$foo_$ver.trim.3ds` on the cartridge and save it
    as a decrypted `.3ds` on the sdcard
- dumping an downloaded title
  - <https://citra-emu.org/wiki/dumping-installed-titles/>
  - `GodMode9` can dump the title to `.cxi` on the sdcard
- dumping updates/dlcs
  - <https://citra-emu.org/wiki/dumping-updates-and-dlcs/>
  - updates/dlcs are distributed as `.cia` archives and unpacked on the
    console
  - `GodMode9` can re-pack them and save them as `.cia` on the sdcard
- <https://github.com/zhaowenlan1779/threeSD>
  - it allows `GodMode9` to dump raw/encrypted data from the console to the
    sdcard and does all processing on the pc
