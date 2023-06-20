Kernel ALSA
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

## AC'97

- Audio Codec '97, developed by intel and codec suppliers in 1997
  - DC (digital controller) is typically in the southbridge
  - AC-LINK is the interface between DC and codecs
  - codecs have AC-LINK connectors on one side and analog audio connectors on
    the other side
- there are many different drivers
  - it standardizes DC-codec interface but not CPU-DC interface
- `CONFIG_SND_INTEL8X0` is for Intel 82801AA and others
  - qemu and crosvm emulate Intel 82801AA, dev id 0x2415
  - it selects `CONFIG_SND_AC97_CODEC`
  - it registers a pci driver `intel8x0_driver`
  - it seems
    - `snd_ac97_bus` creates a `snd_ac97_bus` for the DC
    - `snd_ac97_mixer` creates a `snd_ac97` for each codec

## HD Audio

- Intel High Definition Audio, developed by intel and codec suppliers in 2004
  - host controller is on the pci bus and bridges cpu and codecs
    - suppliers include intel, nvidia, etc.
  - codecs sit behind the host controller
    - suppliers include realtek, etc
  - `CPU <-> Controller <-> Codecs <-> Inputs/Outputs`
- `snd_intel_dsp_driver_probe` determines which driver to use
  - if not intel or before skylake, `SND_INTEL_DSP_DRIVER_ANY`
  - if intel skylake+ without dsp, `SND_INTEL_DSP_DRIVER_LEGACY`
  - otherwise, it uses a table to determine the driver
    - older chromebooks use `SND_INTEL_DSP_DRIVER_SST`
    - the rest uses `SND_INTEL_DSP_DRIVER_SOF` if it has dmic or soundwire
    - there is also `SND_INTEL_DSP_DRIVER_AVS` that is not auto-detected?
  - `azx_driver` is the original driver
    - `azx_probe` continues only if `SND_INTEL_DSP_DRIVER_ANY` or
      `SND_INTEL_DSP_DRIVER_LEGACY`
  - `snd_sof_pci_intel_tgl_driver` is the new driver for tigerlake
    - `hda_pci_intel_probe` continues only if `SND_INTEL_DSP_DRIVER_ANY` or
      `SND_INTEL_DSP_DRIVER_SOF`
  - this section is about the original driver
- `module_hda_codec_driver` registers a codec driver
  - it calls `__hda_codec_driver_register` to register a `hda_codec_driver` on
    `snd_hda_bus_type` bus
- `azx_probe_codecs` is called from the host controller drivers
  - it enumerates codecs connected to the host controller
  - it calls `snd_hda_codec_new` for each codec
    - this adds them as devices on the hda bus for driver binding

## ASoC

- <https://docs.kernel.org/sound/soc/index.html>
  - a machine/board driver glues together platform drivers and codec drivers
  - a platform driver targets an soc cpu and must have no machine-specific
    code
  - a codec driver targets a codec and must have no machine-specific or
    platform-specific code
