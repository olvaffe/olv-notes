Kernel ASoC Intel
=================

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
- there are 3 sets of drivers
  - `snd_intel_dsp_driver_probe` determines which driver to use
    - `SND_INTEL_DSP_DRIVER_ANY` means any driver
  - `SND_INTEL_DSP_DRIVER_LEGACY` means the original non-asoc `azx_driver`
    - this is used unless the platform uses dmic/i2s/soundwire
  - `SND_INTEL_DSP_DRIVER_SST` means the SST drivers
    - this is used with closed-source firmware
    - `sst_acpi_driver` for bay trail, cherry trail
      - this is unmaintained and sof firmware/driver is recommended instead
    - `catpt_acpi_driver` for braswell, broadwell
      - this is active
    - `kmb_plat_dai_driver` for keembay
      - this is active
    - `skl_driver` for skylake to comet lake
      - this is active but is being replaced by the AVS driver
  - `SND_INTEL_DSP_DRIVER_AVS` is a rewrite of SST `skl_driver`
    - `avs_pci_driver` for skylake to comet lake
      - this is active and will replace `sky_driver`
  - `SND_INTEL_DSP_DRIVER_SOF` means the SOF drivers
    - this is used with SOF firmware
    - `snd_sof_acpi_intel_byt_driver` for bay trail, cherry trail
    - `snd_sof_pci_intel_tng_driver` for merrifield
    - `snd_sof_acpi_intel_bdw_driver` for broadwell
    - `snd_sof_pci_intel_skl_driver` for skylake, kaby lake
    - `snd_sof_pci_intel_apl_driver` for apollo lake, gemini lake
    - `snd_sof_pci_intel_cnl_driver` for cannon lake, comet lake
    - `snd_sof_pci_intel_icl_driver` for ice lake
    - `snd_sof_pci_intel_tgl_driver` for tiger lake, alder lake, raptor lake
    - `snd_sof_pci_intel_mtl_driver` for meteor lake
    - `snd_sof_pci_intel_lnl_driver` for lunar lake
- how different dirvers work
  - using apollo lake  / `INT343A` as an example
  - `skl_driver`
    - `skl_probe` calls `skl_find_machine` with
      `snd_soc_acpi_intel_bxt_machines` to find the machine
      - `snd_soc_acpi_find_machine` is used to match by acpi id
      - machine `INT343A` is matched
    - `skl_machine_device_register` creates the platform device
      `bxt_alc298s_i2s`
    - `broxton_audio` drives the platform device
  - `avs_pci_driver`
    - `avs_register_all_boards` calls `avs_register_i2s_boards` with
      `avs_apl_i2s_machines` to register the machine
      - machine `INT343A` is registered
    - `avs_register_i2s_board` creates the platform device `avs_rt298`
    - `avs_rt298_driver` drives the platform device
  - `snd_sof_pci_intel_apl_driver`
    - `hda_pci_intel_probe` calls `sof_pci_probe` with `bxt_desc` as the
      driver data
      - `hda_machine_select` calls `snd_soc_acpi_find_machine` to match by
        acpi id
      - machine `INT343A` is matched
    - `sof_machine_register` creates the platform device `bxt_alc298s_i2s`
    - `broxton_audio` drives the platform device
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
        - `hda_init_caps`
          - `hda_dsp_ctrl_init_chip` calls `hda_codec_detect_mask` to init
            `bus->codec_mask`
          - `hda_codec_probe_bus` probes codecs and loads codec drivers
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
- my brya has devid 0x51c8
  - `adl_desc` is used
  - there are also these acpi devices
    - `10508825:00`
    - `MX98360A:00`
  - `snd_soc_acpi_find_machine` picks `adl_mx98360a_8825` from
    `snd_soc_acpi_intel_adl_machines`
