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
