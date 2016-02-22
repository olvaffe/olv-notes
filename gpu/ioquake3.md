ioquake3
========

## Build System

* Components
  * `ioq3ded` is the dedicated server
  * `ioquake3` is the game engine
  * `qagame<arch>.so` is the server (in charge of everything except rendering)
  * `cgame<arch>.so` is the client (in charge of rendering)
  * `ui<arch>.so` is the game menu
  * A server may run remotely (by `ioq3ded`) or locally (by `ioquake3`)
* The game engine has the idea of multiple games/mods
  * `fs_basepath` is the path to the directory holding all games
  * `fs_homepath` is the path to the directory for all write access
  * `fs_basegame` is the name of the base game (e.g. `baseq3`)
  * `fs_game` is the name of the current game (e.g. a mod)
  * when game data are searched, they are serached in this order
    * fs_homepath/fs_game/pk3s
    * fs_basepath/fs_game/pk3s
    * fs_homepath/fs_basegame/pk3s
    * fs_basepath/fs_basegame/pk3s
    * fs_homepath/BASEGAME/pk3s (compile-time base game)
    * fs_basepath/BASEGAME/pk3s (compile-time base game)
    * see `FS_InitFilesystem`
* ioquake3 requires game data for BASEGAME to be available unless in
  stand-alone mode
  * it will also connect to a server to validate the CD key of BASEGAME
  * thus stand-alone means "not (or not a mod of) Quake III"
* A .qvm is sandboxed, cross-platform, and intepreted shared library
* the sources of `qagame<arch>.so` are mainly
  * `game/*`
* the sources of `cgame<arch>.so` are mainly
  * `cgame/*`
  * `game/bg_*`
* the sources of `ui<arch>.so` are mainly
  * `ui/*`
* the rest sources are for the game engine

## `ioquake3`

* SDL is always initialize unless `NOKIA` is defined.  It also assumes GLES
  instead of GL.
* the entry point is defined in `sys/sys_main.c`
