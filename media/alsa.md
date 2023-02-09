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
    - this seems to scan `/dev/snd` which requires `audio` group
  - `-L` lists all pcm devices including userspace-defined ones
    - this seems to scan `/etc/alsa/conf.d` and add the userspace-defined ones
- `amixer`
  - `controls` lists hw controls
    - this calls `snd_hctl_open`, which is `snd_ctl_open` with caching
    - "High level control interface caches the accesses to primitive controls
      to reduce overhead accessing the real controls in kernel drivers"
  - `scontrols` lists simple controls
    - this calls `snd_mixer_open`
    - "Mixer interface is designed to access the abstracted mixer controls.
      This is an abstraction layer over the hcontrol layer"
- `aplay -D plughw:0,0,0 /usr/share/sounds/sound-icons/prompt.wav`
  - according to `alsactl dump-cfg`, there is a `pcm.plughw`
  - `0,0,0` sets `CARD`, `DEV`, and `SUBDEV` to 0
  - when `0,0,0` is omitted, they default to
    - `defaults.pcm.card`
    - `defaults.pcm.device`
    - `defaults.pcm.subdievce`
- `aplay -D pulse /usr/share/sounds/sound-icons/prompt.wav`
  - `pcm.pulse` is of `pulse` type and uses `libasound_module_pcm_pulse.so`
  - it is configured by `/etc/alsa/conf.d/50-pulseaudio.conf`
- `aplay -D default /usr/share/sounds/sound-icons/prompt.wav`
  - according to `alsactl dump-cfg`, it maps to `cards.pcm.default`
  - by default, `cards.pcm.default` uses a slave of type `plug`, which is
    similar to `pcm.plughw`
- `aplay -D plug:dmix`
  - `pcm.plug` with `dmix` as the slave
  - this uses the `dmix` mixer which can have multiple active clients
