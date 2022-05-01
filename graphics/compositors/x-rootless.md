X Rootless
==========

## Input Driver

- `device_added` in `config/udev.c`
  - `udevadm info /dev/input/event3`
    - `DEVNAME=/dev/input/event3`
    - `/devices/LNXSYSTM:00/LNXPWRBN:00/input/input3/event3`
    - `ID_INPUT=1`
    - `MAJOR=13`
    - `MINOR=67`
    - parent's
      - `PRODUCT=19/0/1/0`
      - name is 'Power Button'
  - `NewInputDeviceRequest`
  - `xf86NewInputDevice`
  - `xf86LoadInputDriver`
  - `systemd_logind_take_fd`
  - sets `XI86_SERVER_FD`

## Video Driver

- `InitOutput`
  - `xf86BusProbe`
    - `xf86platformProbe`
    - `config_odev_probe`
    - `config_udev_odev_probe`
      - `udevadm info /dev/dri/card0`
        - `DEVNAME=/dev/dri/card0`
        - `/devices/pci0000:00/0000:00:02.0/drm/card0`
        - `SUBSYSTEM=drm`
        - `MAJOR=226`
        - `MINOR=0`
        - `ID_PATH=pci-0000:00:02.0`
    - `xf86PlatformDeviceProbe`
    - `get_drm_info`
      - calls `systemd_logind_take_fd` to get an fd
      - sets `XF86_PDEV_SERVER_FD`
  - `xf86OpenConsole`
    - `xf86Info.vtno` is from cmdline and `KeepTty` is true
      - `startx -keeptty vt1`
    - `VT_SETMODE(VT_PROCESS)` such that Xorg gets notified for VT switch
    - `KDSETMODE(KD_GRAPHICS)` such that kernel vt does not draw
    - `KDSKBMODE(K_OFF)` such that kernel vt ignores most keyboard events
    - `tcsetattr` such that kernel ldisc is "raw" (no ISIG, no ICANON, no
      ECHO)
  - `xf86BusConfig`
    - `xf86CallDriverProbe`
    - `xf86platformProbeDev` 
    - `probeSingleDevice`
    - `doPlatformProbe`
    - `ms_platform_probe`

## logind

- `GetSessionByPID` to get the session object of the session Xorg lives in
- `TakeControl` to become the session controller which has exclusive access
  control of managed devices of the session
- `TakeDevice` to get the fd of a managed char device
- `PauseDevice` and `ResumeDevice` siginals to know if a input or output
  device can still be accessed
  - Xorg already has a hand-off mechanism because of `VT_PROCESS`

## startx, xinit, and Xorg

- `startx` is a script that invokes `xinit`
  - it looks into `/tmp/.X11-unix` to decide the available display number
  - it picks `$HOME/.xinitrc` or `/etc/X11/xinit/xinitrc` as the client
  - it picks `$HOME/.xserverrc` or `/etc/X11/xinit/xserverrc` as the server
  - if `tty` prints `/dev/tty[0-9]+`, it adds `vt<N> -keeptty` to server args
  - it invokes `xauth` to set up the server auth file at
    `/tmp/serverauth.XXXXXXXXXX` and adds `-auth /tmp/serverauth.XXXXXXXXXX`
    to server args
  - it sets `XAUTHORITY=$HOME/.Xauthority` and invokes `xauth` again to add
    the same credential to the file; this allows the user to connect to the
    server
  - it invokes `xinit` like this
    - `xinit $HOME/.xinitrc -- /etc/X11/xinit/xserverrc :0 vt1 -keeytty -auth /tmp/serverauth.hDjnusR84x`
  - after xinit returns, it invokes `xauth remove :0` to remove the credential
- `xinit` starts both the server and the client
  - it forks and execs the server specified by `startx`, normally
    `/etc/X11/xinit/xserverrc`
  - it forks again and execs the client specified by `startx`, normally
    `$HOME/.xinitrc`
  - when the client process exits, it kills the server process
    - it send `SIGTERM` to server and prints `waiting for X server to shutdown`
    - server prints `Server terminated successfully (0). Closing log file.`
      and exits
- `Xorg` is the server
  - `xinit` normally starts `/etc/X11/xinit/xserverrc` as the server, which is
    a script that does
    - `exec /usr/bin/X -nolisten tcp "$@"`
  - `/usr/bin/X` is a symlink to `/usr/bin/Xorg`
  - `/usr/bin/Xorg` is a script that exec `/usr/lib/Xorg.wrap` or
    `/usr/lib/Xorg`
  - `/usr/lib/Xorg.wrap` is a suid wrapper to `/usr/lib/Xorg`
    - it checks `/etc/X11/Xwrapper.config` or autodtect, and decide
      - whether the user can start the server or not
      - whether the user needs root to start the server or not
    - starting the server in ssh gives an error thanks to `Xorg.wrap`
      - `/usr/lib/Xorg.wrap: Only console users are allowed to run the X server`
  - `/usr/lib/Xorg` is the real server
    - starting the server in ssh still gives an error
      - `xf86OpenConsole: Cannot open virtual console 1 (Permission denied)`
      - this is because, when starting in tty, logind chowns /dev/tty1 for the
      	server
      - `TakeControl` method of `org.freedesktop.login1.Session`
