# Linux ALSA USB

## Initialization

- `usb_audio_probe` probes `USB_CLASS_AUDIO` usb audio devices
- `snd_usb_audio_create` creates a `snd_usb_audio` for a usb device
  - `snd_card_new` creates `chip->card`
  - `snd_usb_audio_create_proc` creates `/proc/asound/<card>/{usbbus,usbhid}`
- `snd_usb_create_streams` calls `snd_usb_create_stream` for usb interfaces
  - if midi keyboard, `snd_usb_midi_v2_create`
    - it falls back to `__snd_usbmidi_create` for midi 1.0
    - `snd_usbmidi_create_rawmidi` creates a `snd_rawmidi`
    - `snd_usbmidi_create_endpoints` creates `snd_usb_midi_in_endpoint` for
      inputs and `snd_usb_midi_out_endpoint` for outputs
      - `snd_usbmidi_init_substream` inits the substreams (ports) of the
        endpoints
  - if speaker, mic, etc.,
    - `snd_usb_parse_audio_interface` parses an interface
    - `snd_usb_add_audio_stream` calls `snd_pcm_new`
    - `snd_usb_add_endpoint` adds an `snd_usb_endpoint`
- `snd_usb_create_mixer`
  - `snd_usb_mixer_controls` creates `usb_mixer_elem_info`s and wraps them in
    `snd_kcontrol`s
  - it calls `snd_device_new` with `SNDRV_DEV_CODEC` to wrap the
    `usb_mixer_interface`
- `try_to_register_card`
