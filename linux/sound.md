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
    - suppliers include intel, amd, nvidia, via, creative, etc.
  - codecs sit behind the host controller
    - suppliers include realtek, etc
  - `CPU <-> Controller <-> Codecs <-> Inputs/Outputs`
- `snd_intel_dsp_driver_probe` determines which driver to use
  - <https://thesofproject.github.io/latest/getting_started/intel_debug/introduction.html>
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
- a machine driver defines a `struct snd_soc_card`
  - the machine driver calls `devm_snd_soc_register_card` to register its
    `snd_soc_card`
- codec drivers are under `sound/soc/codecs`
  - each codec driver must define a `snd_soc_dai_driver`, which is a means to
    communicate with the codec

## ASoC: Case Study

- take `CONFIG_SND_SOC_AMD_ACP6x` for example
- `yc_acp6x_driver` binds to pci device with devid `ACP_DEVICE_ID`
  - `snd_acp6x_probe` probes the pci device and registers 3 platform devices
    - `acp_yc_pdm_dma`
    - `dmic-codec`
    - `acp_yc_pdm_dma`
- `acp6x_pdm_dma_driver` binds to `acp_yc_pdm_dma`
  - this is an asoc platform driver
  - `acp6x_pdm_audio_probe` calls `devm_snd_soc_register_component` to
    register `acp6x_pdm_component` and `acp6x_pdm_dai_driver`
- `dmic_driver` binds to `dmic-codec`
  - this is an asoc codec driver
  - `dmic_dev_probe` calls `devm_snd_soc_register_component` to register
    `soc_dmic` and `dmic_dai`
- `acp6x_mach_driver` binds to `acp_yc_mach`
  - this is an asoc machine driver
  - `acp6x_probe` calls `devm_snd_soc_register_card` to register `acp6x_card`
  - `acp_dai_pdm` is the only `snd_soc_dai_link` of the card
    - `acp_pdm` is the cpu component of the link
    - `dmic_codec` is the codec component of the link
    - `platform` is the platform component of the link
  - as a part of card registration, `snd_soc_add_pcm_runtime` processes the
    dai link
    - it calls `soc_new_pcm_runtime` to create a `snd_soc_pcm_runtime`
    - it looks up all cpu components, codec components, and platform
      components and adds them to the `snd_soc_pcm_runtime`
- it seems
  - the physical link is `ACP <- DMIC <- input`
  - the asoc representation is `CPU DAI <- CODEC DAI`
  - the data flow is `mem <- CPU DAI <- CODEC DAI <- input`
    - that is, all components on the link have DAIs to connect them together
    - the CPU DAI is different from CODEC DATA in that it additionally
      supports DMA to write the input data to the system memory

## ASoC: Intel

- Intel SST (Intel Smart Sound Technology) is a DSP
  - sst is the original driver for the dsp
  - avs (Audio-Voice-Speech) is the new driver for the dsp
- the devid is 0xa0c8 on my tigerlake
  - both `snd_sof_pci_intel_tgl_driver` and `azx_driver` support the device
  - because it is connected to DMIC on my machine,
    `snd_intel_dsp_driver_probe` picks `SND_INTEL_DSP_DRIVER_SOF`
  - `snd_sof_pci_intel_tgl_driver` uses `tgl_desc` as the device description
    - `tgl_desc` supports `SOF_IPC` and `SOF_INTEL_IPC4`
    - `SOF_IPC` uses the current `sof-tgl.ri` firmware
    - `SOF_INTEL_IPC4` uses the newer `dsp_basefw.bin` firmware
- after `sof-audio-pci-intel-tgl` binds to my `0xa0c8`, it creates multiple
  subdevices
  - `dmic-codec` is on the platform bus and needs `dmic-codec`
  - `ehdaudio0D0` is on the hdaudio bus and needs `snd_hda_codec_realtek`
  - `ehdaudio0D2` is on the hdaudio bus and needs `snd_hda_codec_hdmi`
  - `skl_hda_dsp_generic` is on the platform bus and needs `skl_hda_dsp_generic`
  - configs
    - `CONFIG_SND_SOC_SOF_TIGERLAKE`
    - `CONFIG_SND_SOC_DMIC`
    - `CONFIG_SND_HDA_CODEC_REALTEK`
    - `CONFIG_SND_HDA_CODEC_HDMI`
    - `CONFIG_SND_SOC_INTEL_SKL_HDA_DSP_GENERIC_MACH`
      - depends on `CONFIG_SND_SOC_INTEL_SKL` and
        `CONFIG_SND_SOC_INTEL_SKYLAKE_HDAUDIO_CODEC`

## ASoC: AMD

- all the soc pci drivers match `PCI_DEVICE(PCI_VENDOR_ID_AMD, 0x15e2)`
  - `git grep module_pci_driver sound/soc/{amd,sof/amd}` lists all of them
  - `snd_amd_acp_find_config` determines which driver to use
    - if chromebook, return `FLAG_AMD_SOF`
    - else, return 0
    - `FLAG_AMD_LEGACY` is never set
  - these drivers require `FLAG_AMD_SOF`
    - `snd_sof_pci_amd_rmb_driver`
    - `snd_sof_pci_amd_rn_driver`
  - these drivers require `FLAG_AMD_LEGACY`
    - `snd_amd_acp_pci_driver`
  - these drivers require 0
    - `acp3x_driver` is for rev 0x00
    - `rn_acp_driver` is for rev 0x01
    - `acp5x_driver` is for rev 0x50
    - `yc_acp6x_driver` is for rev 0x60 and 0x6f
    - `rpl_acp6x_driver` is for rev 0x62
    - `ps_acp63_driver` is for rev 0x63
- on my renoir,
  - `snd_hda_intel` binds to the hda device
    - `CONFIG_SND_HDA_INTEL`
    - because the driver has `PCI_DEVICE(PCI_VENDOR_ID_AMD, PCI_ANY_ID)`
    - it creates a subdevice `hdaudioC0D0` whose driver is
      `snd_hda_codec_hdmi`
      - `CONFIG_SND_HDA_CODEC_HDMI`
  - `snd_sof_amd_renoir` binds to `0x15e2`
    - `CONFIG_SND_SOC_SOF_AMD_RENOIR`
    - it creates multiple subdevices
      - `dmic-codec` whose driver is `dmic-codec`
        - `CONFIG_SND_SOC_DMIC`
      - `rt5682s-rt1019` whose driver is `sof_mach`
        - `CONFIG_SND_SOC_AMD_SOF_MACH`
