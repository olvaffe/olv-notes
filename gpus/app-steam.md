Steam
=====

## Directory Structure

- `steam_latest.deb`
  - it installs `/usr/bin/steam` which is a symlink to
    `/usr/lib/steam/bin_steam.sh`
  - `setup_variables`
    - `STEAMSCRIPT=/usr/lib/steam/bin_steam.sh`
    - `STEAMPACKAGE=steam`
    - `STEAMCONFIG=~/.steam`
    - `STEAMDATALINK="$STEAMCONFIG/$STEAMPACKAGE"`
    - `LAUNCHSTEAMBOOTSTRAPFILE=/usr/lib/steam/bootstraplinux_ubuntu12_32.tar.xz`
    - `STEAM_DATA_HOME="$HOME/.local/share"`
    - `DEFAULTSTEAMDIR="$STEAM_DATA_HOME/Steam"`
  - `install_bootstrap`
    - untar `bootstraplinux_ubuntu12_32.tar.xz` to `~/.local/share/Steam`
      - this untars `steam.sh`
    - link `~/.steam/steam` to `~/.local/share/Steam`
    - `setup_variables` again
      - `LAUNCHSTEAMDIR=~/.local/share/Steam`
  - `cp $LAUNCHSTEAMBOOTSTRAPFILE $LAUNCHSTEAMDIR/bootstrap.tar.xz`
  - `exec ~/.local/share/Steam/steam.sh`
- `/usr/bin/steam` execs `~/.local/share/Steam/steam.sh`
  - `STEAMROOT=~/.local/share/Steam`
  - `STEAMDATA=~/.local/share/Steam`
  - `STEAMEXE=steam`
  - `PLATFORM=ubuntu12_32`
  - `PLATFORM32=ubuntu12_32`
  - `PLATFORM64=ubuntu12_64`
  - `STEAMEXEPATH=ubuntu12_32/steam`
  - on first launch, it invokes `steamdeps steamdeps.txt` to install dependent
    distro packages
  - `STEAM_RUNTIME=$STEAMROOT/ubuntu12_32/steam-runtime`
    - built from `https://github.com/ValveSoftware/steam-runtime`
    - `setup.sh` is installed by bootstrap
  - it finally invokes `$STEAMROOT/ubuntu12_32/steam`
- `~/.local/share/Steam/ubuntu12_32/steam`
  - is the proprietary steam client
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
