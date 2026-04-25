# Linux ALSA Core

## Overview

- `snd_card` is the top-level container
  - `devices` is a list of `snd_device`
    - `snd_pcm`, etc.
  - `controls` is a list of `snd_kcontrol`
