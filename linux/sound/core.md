# Linux ALSA Core

## Overview

- `snd_card` is the top-level container
  - `devices` is a list of `snd_device`
    - `snd_pcm`, etc.
  - `controls` is a list of `snd_kcontrol`

## MIDI

- `snd_rawmidi` exposes `/dev/snd/midiC0D0` or `/dev/snd/umpC0D0` for raw midi
  data
  - if `CONFIG_SND_SEQUENCER`, it also creates a `snd_seq_device`
- sequencer
  - I guess the sequencer can route midi-like cmds from input ports to output
    ports
  - when sw synthesizers such as `fluidsynth` or `timidity` are in daemon
    mode, it creates one or more output ports
    - their job is to synthesize received cmds using a soundfont and play the
      audio to an output device
  - `playmidi` creates an input port, feeds cmds to the port, and route them
    to the output ports
  - a usb midi keyboard probably creates an input port
    - pressing a key feeds the cmd to the port
