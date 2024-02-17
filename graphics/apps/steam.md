Steam
=====

## Troubleshooting

- `pacman -S steam`
  - `pacman -S ttf-liberation` if ui text is garbled
  - `pacman -S lib32-systemd` if no network

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
- apps are downloaded to `~/.local/share/Steam/steamapps`
- app database
  - the database is at `~/.local/share/Steam/appcache/appinfo.vdf`
  - VDF stands for Valve Data Format
  - `pip install steam` to get a parser
    - or, simply visit `https://steamdb.info/app/<appid>/config/`
  - take appid 570 for example
    - 'installdir': 'dota 2 beta'
    - 'executable': 'game/dota.sh'
    - 'arguments': '+engine_experimental_drop_frame_ticks 1 +@panorama_min_comp_layer_dimension 0 -prewarm_panorama'
- user data
  - `userdata/<userid>/config/localconfig.vdf` contains user configs
  - e.g., custom launch options

## Steam Runtimes

- <https://gitlab.steamos.cloud/steamrt>
- v1, scout
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/scout>
    - <https://gitlab.steamos.cloud/steamrt/scout>
    - <https://steamdb.info/app/1070560/>
    - <https://github.com/ValveSoftware/steam-runtime>
  - v1 is `LD_LIBRARY_PATH`-based
  - manual
    - `~/.local/share/Steam/ubuntu12_32/steam-runtime/run.sh
      ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`
    - `installdir`, `executable`, and `arguments` are from the app database
- v2, soldier
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/soldier>
    - <https://gitlab.steamos.cloud/steamrt/soldier>
    - <https://steamdb.info/app/1391110/>
  - v2 is container-based
    - debian 10
    - for use with proton 5.13, 6.3, and 7.0
  - manual
    - `~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_soldier/run
      ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`
- v3, sniper
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/sniper>
    - <https://gitlab.steamos.cloud/steamrt/sniper>
    - <https://steamdb.info/app/1628350/>
  - v2 is also container-based
    - debian 11
    - for use with proton 8.0

## Proton

- proton is installed as apps
  - since proton 5.13, it should be run under steam runtime v2
  - since proton 8.0, it shoud be run under steam runtime v3
- 7.0, <https://steamdb.info/app/1887720/>
- 8.0, <https://steamdb.info/app/2348590/>
- manual
  - `STEAM_COMPAT_CLIENT_INSTALL_PATH=~/.local/share/Steam
     STEAM_COMPAT_DATA_PATH=~/.local/share/Steam/steamapps/compatdata/<appid>
     ~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_soldier/run
     ~/.local/share/Steam/steamapps/common/Proton\ -\ Experimental/proton
     waitforexitandrun
     ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`

## Using runtime and proton directly

- use runtime directly
  - `cp -a ~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_sniper .`
  - parse runtime vdf for cmdline
    - `apt install python3-vdf`
    - `cp /usr/share/doc/python3-vdf/examples/vdf2json .`
    - `./vdf2json SteamLinuxRuntime_sniper/toolmanifest.vdf tmp`
    - we will get `/_v2-entry-point --verb=%verb% --`
  - run any cmd
    - `./SteamLinuxRuntime_sniper/_v2-entry-point --verb=waitforexitandrun -- ls`
- use proton under runtime directly
  - `cp -a ~/.local/share/Steam/steamapps/common/Proton\ 8.0 .`
  - parse proton vdf for cmdline
    - we will get `/proton %verb%`
  - prepare env
    - mkdir `prefix`
    - `export STEAM_COMPAT_DATA_PATH=$PWD/prefix`
    - `export DXVK_STATE_CACHE_PATH=$PWD/prefix`
    - `export STEAM_COMPAT_MOUNTS=`, no additional bind-mount
    - `export STEAM_COMPAT_CLIENT_INSTALL_PATH=`
    - `export PROTON_LOG=1`
  - run any cmd
    - `./SteamLinuxRuntime_sniper/_v2-entry-point --verb=waitforexitandrun -- $PWD/Proton\ 8.0/proton waitforexitandrun notepad`
