ALSA Userspace
==============

## Repos

- <https://git.alsa-project.org/>
- `alsa-plugins` are plugins
  - plugins are installed as `/usr/lib/alsa-lib/libasound_module_*.so`
  - plugin configs are installed as `/usr/share/alsa/alsa.conf.d/*.conf`
- `alsa-utils` are various utilities
- `alsa-lib` provides card configs and C libraries/headers

## procfs

- `/proc/asound/modules` lists kernel modules of all cards
  - format is `<card> <module-name>`
- `/proc/asound/cards` lists all cards
  - format is `<card> [<id>]: <driver> <short-name> <long-name>`
- a card can have multiple devices
- `/proc/asound/hwdep` lists hwdep devices of all cards
  - format is `<card>-<dev>: <name>`
- `/proc/asound/hwdep` lists hwdep devices of all cards
  - format is `<card>-<dev>: <name>`
- `/proc/asound/pcm` lists pcm devices of all cards
  - format is `<card>-<dev>: <id> : <name> : playback <stream-count> : capture <stream-count>`
- `/proc/asound/timer` lists timers
  - format is `G<idx> : <name>` if a `SNDRV_TIMER_CLASS_GLOBAL`
  - format is `C<card>-<dev> : <name>` if a `SNDRV_TIMER_CLASS_CARD`
  - format is `P<card>-<dev>-<subdev> : <name>` if a `SNDRV_TIMER_CLASS_PCM`
- `/proc/asound/devices` lists all char devices under `/dev/snd`
  - format is `<minor>: [<card>-<dev>]: <type>`
  - `card` and `dev` are optional
    - e.g., `/dev/snd/seq` has minor 1 and no card
  - `type` is commonly
    - `sequencer`
    - `digital audio capture`
    - `digital audio playback`
    - `hardware dependent`
    - `control`
    - `timer`
- when `/proc/asound/card0` is HDA
  - hwdep devices may provide codecs and/or elds (EDID-like data)
    - `codec#<dev>`
    - `eld#<dev>.<subdev>` provides EDID when connected to a monitor
  - `pcm<dev>c` for capture
  - `pcm<dev>p` for playback

## Userspace

- `aplay`
  - `-l` lists hw pcm devices
  - `-L` lists all pcm devices including userspace-defined ones
- `amixer`
  - `controls` lists hw controls
    - this calls `snd_hctl_open`, which is `snd_ctl_open` with caching
    - "High level control interface caches the accesses to primitive controls
      to reduce overhead accessing the real controls in kernel drivers"
  - `scontrols` lists simple controls
    - this calls `snd_mixer_open`
    - "Mixer interface is designed to access the abstracted mixer controls.
      This is an abstraction layer over the hcontrol layer"
