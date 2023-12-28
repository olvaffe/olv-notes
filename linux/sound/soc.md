Kernel asoc
===========

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

## ASoC: AMD

- AMD Audio Coprocessor (ACP)
  - a pci device with `PCI_DEVICE(PCI_VENDOR_ID_AMD, ACP_PCI_DEV_ID)` (0x15e2)
  - different revisions require different drivers
    - raven has rev 0x00 which is driven by `acp3x_driver`
    - renoir has rev 0x01 which is driven by `rn_acp_driver`
    - vangogh has rev 0x50 which is driven by `acp5x_driver`
    - yellow carp (rembrandt) has rev 0x60 or 0x6f which is driven by
      `yc_acp6x_driver`
    - raphael has rev 0x62 which is driven by `rpl_acp6x_driver`
    - pink sardine (phoenix) has rev 0x63 which is driven by `ps_acp63_driver`
  - even the same revision can have different firmwares and require different
    drivers
    - `snd_amd_acp_find_config` can return
      - `FLAG_AMD_SOF`, which means acp uses the sof firmware
        - renoir is driven by `snd_sof_pci_amd_rn_driver` instead
        - vangogh is driven by `snd_sof_pci_amd_vgh_driver` instead
        - rembrandt is driven by `snd_sof_pci_amd_rmb_driver` instead
        - phoenix is driven by `snd_sof_pci_amd_acp63_driver` instead
      - `FLAG_AMD_LEGACY`, which means acp uses the legacy firmware(?)
        - all supported revisions are driven by `snd_amd_acp_pci_driver`
          instead, which creates different platform devices for different
          revisions
        - renoir is driven by `renoir_driver` instead
        - rembrandt is driven by `rembrandt_driver` instead
        - phoenix  is driven by `acp63_driver` instead
        - revision 0x70 is driven by `acp70_driver` instead
- how different drivers work
  - the default drivers usually power on acp, detect the config, and register
    config-dependent platform devices, and let platform drivers take over
    - for example, on rembrandt, `acp6x_mach_driver` and others take over
  - the `FLAG_AMD_SOF` drivers usually call the generic `sof_pci_probe` with
    driver-dependent `sof_dev_desc`
    - for example, on rembrandt, `snd_sof_pci_amd_rmb_driver` calls
      `sof_pci_probe` with `snd_soc_acpi_amd_rmb_sof_machines`
    - if `RTL5682` is selected, `sof_pci_probe` creates `rt5682s-hs-rt1019`
      platform device which binds to `sof_mach` platform driver
  - the `FLAG_AMD_LEGACY` drivers usually register platform devices and let
    platform drivers take over
    - for example, on rembrandt, `rembrandt_driver` calls `acp_machine_select`
      with `snd_soc_acpi_amd_rmb_acp_machines`
    - if `RTL5682` is selected, `acp_machine_select` creates
      `rmb-rt5682s-rt1019` platform device which binds to `acp_mach` platform
      driver
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
