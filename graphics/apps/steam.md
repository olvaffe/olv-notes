Steam
=====

## Official `steam_latest.deb`

- `/usr/bin/steam` is a symlink to `/usr/lib/steam/bin_steam.sh`
  - it bootstraps `~/.local/share/Steam`
    - `install_bootstrap` untars
      `/usr/lib/steam/bootstraplinux_ubuntu12_32.tar.xz` to the directory
    - the tarball is also copied over (and can be used to re-bootstrap)
  - it creates `~/.steam`
    - `~/.steam/steam` is a symlink to `~/.local/share/Steam`
  - it runs `~/.local/share/Steam/steam.sh`
- `/usr/bin/steam` execs `~/.local/share/Steam/steam.sh`
  - it checks host package dependencies
    - `/usr/bin/steamdeps ~/.local/share/Steam/steamdeps.txt`
  - it creates various symlinks in `~/.steam`
  - it sets up steam runtime
    - `STEAM_RUNTIME=~/.local/share/Steam/ubuntu12_32/steam-runtime`
    - the runtime is unpacked from the bootstrap tarball
      - source at <https://github.com/ValveSoftware/steam-runtime>
    - `setup.sh` uses `zenity` to display a `Updating Steam runtime
      environment...` dialog
  - it sets up `LD_LIBRARY_PATH` to use the libraries from the runtime
    - with the help of
      `~/.local/share/Steam/ubuntu12_32/steam-runtime/run.sh --print-steam-runtime-library-paths`
  - it finally invokes `~/.local/share/Steam/ubuntu12_32/steam`
- `~/.local/share/Steam/ubuntu12_32/steam`
  - it is the proprietary steam client
  - if `DEBUGGER=gdb` is set, it is run under gdb
  - it downloads/updates all packages
    - bootstrap is a minimal environment to run the client
    - the client downloads more packages to `~/.local/share/Steam/package`

## steam client

- for command line options
  - <https://developer.valvesoftware.com/wiki/Command_Line_Options#Steam_.28Windows.29>
  - `-silent` starts steam client in the background
  - `-shutdown` kills steam client
  - `-login`
  - `-install`
  - `-console`
  - `-applaunch`

## Run a Game

- `appcache/appinfo.vdf` contains all app info
  - VDF stands for Valve Data Format
  - `pip install steam` to get a parser
  - or, simply visit `https://steamdb.info/app/<appid>/config/`
  - take appid 570 for example
    'installdir': 'dota 2 beta'
    'executable': 'game/dota.sh'
    'arguments': '+engine_experimental_drop_frame_ticks 1 +@panorama_min_comp_layer_dimension 0 -prewarm_panorama'
- `userdata/<userid>/config/localconfig.vdf` contains user configs
  - e.g., launch options
- BELOW IS UNVERIFIED
- to run a game using steam runtime 1 (scout)
    ~/.steam/steam/ubuntu12_32/steam-runtime/run.sh \
      ~/.steam/steam/steamapps/common/<installdir>/<executable> <arguments>
- to run a game using steam runtime 2 (soldier)
    ~/.steam/steam/stramapps/common/SteamLinuxRuntime_soldier/_v2-entry-point
      ~/.steam/steam/steamapps/common/<installdir>/<executable> <arguments>
- to run a game using proton 5.0
    STEAM_COMPAT_DATA_PATH=~/.steam/steam/steamapps/compatdata/<appid> \
    ~/.steam/steam/stramapps/common/Proton\ 5.0/proton waitforexitandrun \
      ~/.steam/steam/steamapps/common/<installdir>/<executable> <arguments>
- to run a game using proton 5.13
  - run the game under proton under soldier?

## steamrt

- <https://gitlab.steamos.cloud/steamrt>

## proton

- `cd /.local/share/Steam/steamapps/common/Proton\ - \ Experimental`
- `proton` is a python script
  - `STEAM_COMPAT_CLIENT_INSTALL_PATH=~/.local/share/Steam`
  - `STEAM_COMPAT_DATA_PATH`
