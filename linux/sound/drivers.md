Linux sound
===========

## Generic Drivers

- `CONFIG_SND_DUMMY` is for a null device
  - `alsa_card_dummy_init` registers a platform driver, `snd_dummy_driver`
    whose name is `SND_DUMMY_DRIVER` (`snd_dummy`), and creates a dummy
    platform device for it
- `CONFIG_SND_PCSP` is for pc speaker
  - it replaces `CONFIG_INPUT_PCSPKR`
  - `pcsp_init` registers a platform driver, `pcsp_platform_driver` whose name
    is `pcspkr`
  - it talks to the hw via io port 0x61
