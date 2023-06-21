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
- `azx_driver` is the pci driver
  - it probes any pci device of `PCI_CLASS_MULTIMEDIA_HD_AUDIO`
  - `snd_intel_dsp_driver_probe` determines if this driver should be used
    - this driver only supports proper HDA controllers
    - if non-intel (amd, nvidia, etc.), the device is a proper HDA controller
    - if intel, the device might have a dsp and this driver might not work
      - skylake+ has a dsp but it may or may not be enabled
      - certain older atoms has a dsp as well
- `azx_probe_codecs` is called from the host controller drivers
  - it enumerates codecs connected to the host controller
  - it calls `snd_hda_codec_new` for each codec
    - this adds them as devices on the hda bus for driver binding
- `module_hda_codec_driver` registers a codec driver
  - it calls `__hda_codec_driver_register` to register a `hda_codec_driver` on
    `snd_hda_bus_type` bus
  - note that even when `azx_driver` is not used, other hda drivers still have
    their ways to discover codecs on the hda bus and these codec drivers are
    reused

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
  - note that it only supports capture
  - playback on amd devices often goes through the HDA controller
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

- intel aDSP (audio DSP)
  - <https://thesofproject.github.io/latest/getting_started/intel_debug/introduction.html>
  - <https://docs.zephyrproject.org/latest/boards/xtensa/intel_adsp_cavs25/doc/intel_adsp_generic.html>
  - first gen: SST (smart sound technology)
    - Xtensa HiFi2 EP
    - used on bay trail, cherry trail, braswell, and broadwell
    - discovered via acpi
    - can be enabled/disabled in bios
      - when enabled, must use the asoc driver
      - when disabled, the original `azx_driver` can be used
    - does not handle hdmi/dp
  - second gen: cAVS (Converged Audio Voice Speech)
    - Xtensa HiFi3
    - used on
      - cAVS 1.5: skylake, kaby lake, apollo lake, gemini lake
      - cAVS 1.8: cannon lake, whiskey lake, comet lake
      - cAVS 2.0: ice lake
      - cAVS 2.5: tiger lake
    - discovered via pci
    - can be enabled/disabled in bios
      - if a dcim is attached, or if i2c/i2s/soundwire is used, must be
        enabled
  - third gen: ACE (Audio Capture Engine)
    - Xtensa
    - used on
      - ACE 1.5: meteor lake
      - ACE 2.0: lunar lake
  - intel requires the dsp firmware to be signed
    - unless it's chromebook
- `snd_intel_dsp_driver_probe` determines which driver to use
  - `SND_INTEL_DSP_DRIVER_ANY` means any driver
  - `SND_INTEL_DSP_DRIVER_LEGACY` means the original `azx_driver`
  - `SND_INTEL_DSP_DRIVER_SST` means
    - `sst_acpi_driver` for bay trail, cherry trail
    - `catpt_acpi_driver` for braswell, broadwell
    - `skl_driver` for skylake to comet lake on closed-source firmware
  - `SND_INTEL_DSP_DRIVER_AVS` is a rewrite of `skl_driver`
  - `SND_INTEL_DSP_DRIVER_SOF` means any adsp on SoF firmware
    - `snd_sof_acpi_intel_byt_driver` for bay trail, cherry trail
    - `snd_sof_acpi_intel_bdw_driver` for broadwell
    - `snd_sof_pci_intel_skl_driver` for skylake, kaby lake
    - `snd_sof_pci_intel_apl_driver` for apollo lake, gemini lake
    - `snd_sof_pci_intel_cnl_driver` for cannon lake, comet lake
    - `snd_sof_pci_intel_icl_driver` for ice lake
    - `snd_sof_pci_intel_tgl_driver` for tiger lake, alder lake, raptor lake
    - `snd_sof_pci_intel_mtl_driver` for meteor lake
- my tigerlake has devid 0xa0c8
  - `azx_driver` binds, probes, and bails because `snd_intel_dsp_driver_probe`
    picks `SND_INTEL_DSP_DRIVER_SOF`
    - `DSP detected with PCI class/subclass/prog-if info 0x040380`
    - `Digital mics found on Skylake+ platform, using SOF driver`
  - `snd_sof_pci_intel_tgl_driver` calls `hda_pci_intel_probe` to probe and
    succeeds
    - `tgl_desc` is the device description
      - it supports `SOF_IPC` and `SOF_INTEL_IPC4`
      - `SOF_IPC` uses the current `sof-tgl.ri` firmware
      - `SOF_INTEL_IPC4` uses the newer `dsp_basefw.bin` firmware based on
        zephyr rtos
    - `sof_pci_probe` does many initializations.  Among them, these are called
      from `snd_sof_device_probe`
      - `sof_ops_init` calls `sof_tgl_ops_init`
        - it initializes `sof_tgl_ops` from `sof_hda_common_ops`
      - `snd_sof_probe` calls `hda_dsp_probe`
        - this creates the `dmic-codec` platform device
      - `sof_machine_check` calls `snd_sof_machine_select` which calls
        `hda_machine_select`
        - it ends up in `hda_generic_machine_select` and selects
          `snd_soc_acpi_intel_hda_machines` 
  - there are these subdevices and drivers
    - `dmic-codec` is on the platform bus and with `dmic_driver`
    - `ehdaudio0D0` is on the hdaudio bus and with `realtek_driver`
    - `ehdaudio0D2` is on the hdaudio bus and with `hdmi_driver`
    - `skl_hda_dsp_generic` is on the platform bus with `skl_hda_audio`
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
