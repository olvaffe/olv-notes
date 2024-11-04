Kernel ASoC AMD
===============

## ASoC: AMD

- AMD Audio Coprocessor (ACP)
  - a pci device with `PCI_DEVICE(PCI_VENDOR_ID_AMD, ACP_PCI_DEV_ID)` (0x15e2)
    - there might be a separate pci device for an hda controller
      - this hda controller often has a codec for just hdmi
  - different revisions require different drivers
    - raven has rev 0x00 which is driven by `acp3x_driver`
    - renoir has rev 0x01 which is driven by `rn_acp_driver`
    - vangogh has rev 0x50 which is driven by `acp5x_driver`
    - yellow carp (rembrandt) has rev 0x60 or 0x6f which is driven by
      `yc_acp6x_driver`
    - raphael has rev 0x62 which is driven by `rpl_acp6x_driver`
    - pink sardine (phoenix) has rev 0x63 which is driven by `ps_acp63_driver`
  - there is also a newer `snd_amd_acp_pci_driver`
    - it supports multiple revisions in a single pci driver
    - it creates different platform devices for different revisions
      - renoir (0x01) is driven by `renoir_driver` instead
      - rembrandt (0x6f) is driven by `rembrandt_driver` instead
      - phoenix (0x63) is driven by `acp63_driver` instead
      - revision 0x70 and 0x71 are driven by `acp70_driver` instead
  - there are even newer sof drivers, for when the acp runs sof firmware
    - renoir is driven by `snd_sof_pci_amd_rn_driver` instead
    - vangogh is driven by `snd_sof_pci_amd_vgh_driver` instead
    - rembrandt is driven by `snd_sof_pci_amd_rmb_driver` instead
    - phoenix is driven by `snd_sof_pci_amd_acp63_driver` instead
  - `snd_amd_acp_find_config` decides which driver to use
    - no flag uses the old drivers
    - `FLAG_AMD_LEGACY` uses the common `snd_amd_acp_pci_driver`
    - `FLAG_AMD_SOF` uses the sof drivers
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

## ASoC: Rembrandt

- there are 3 drivers
  - `CONFIG_SND_SOC_AMD_ACP6x`
  - `CONFIG_SND_SOC_AMD_ACP_PCI`
  - `CONFIG_SND_SOC_SOF_AMD_REMBRANDT`
- `snd_amd_acp_find_config` determines the driver
  - no flag uses `CONFIG_SND_SOC_AMD_ACP6x`
  - `FLAG_AMD_LEGACY` uses `CONFIG_SND_SOC_AMD_ACP_PCI`
  - `FLAG_AMD_SOF` uses `CONFIG_SND_SOC_SOF_AMD_REMBRANDT`
- `CONFIG_SND_SOC_AMD_ACP6x` is capture-only
  - capture uses PDM-connected DMIC
  - playback goes through another pci device which is an hda controller
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
      - `acp6x_pdm` is the cpu component of the link
      - `dmic_codec` is the codec component of the link
      - `pdm_platform` is the platform component of the link
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
- `CONFIG_SND_SOC_AMD_ACP_PCI` supports both capture and playback
  - capture uses PDM-connected DMIC
  - playback uses I2S-connected codecs
  - `acp_pci_probe` is the generic pci driver and registers these platform
    devies
    - `dmic-codec`
    - `acp_asoc_rembrandt`
  - `rembrandt_audio_probe` probes `acp_asoc_rembrandt`
    - `acp_rmb_dai` is an array of 4 `snd_soc_dai_driver`
      - `acp-i2s-sp` is for SP (speaker?) playback/capture
      - `acp-i2s-bt` is for BT (bluetooth?) playback/capture
      - `acp-i2s-hs` is for HS (headset?) playback/capture
      - `acp-pdm-dmic` is for DMIC capture
    - `snd_soc_acpi_amd_rmb_acp_machines` is an array of 3 `snd_soc_acpi_mach`
    - `acp_machine_select` selects the correct machine and registers a
      platform device
    - `acp_platform_register` calls `devm_snd_soc_register_component` to
      register a component
  - `acp_asoc_probe` probes the machine platform device
    - `acp_legacy_dai_links_create` creates `snd_soc_dai_link`s
    - `devm_snd_soc_register_card` registers a card
- `CONFIG_SND_SOC_SOF_AMD_REMBRANDT` supports both capture and playback, when
  acp runs on sof firmware
  - capture uses PDM-connected DMIC
  - playback uses I2S-connected codecs
  - `acp_pci_rmb_probe`
    - `rembrandt_desc` is the device data
      - `snd_soc_acpi_amd_rmb_sof_machines` is an array of 3
        `snd_soc_acpi_mach`
      - `sof_rembrandt_ops` is the ops
    - `sof_pci_probe` is the standard sof probe
      - `snd_sof_machine_select` calls `amd_sof_machine_select` to select the
        machine
      - `devm_snd_soc_register_component` registers the component
      - `snd_sof_machine_register` calls `sof_machine_register` to register
        the machine as a platform device
  - `acp_sof_probe` probes the machine platform device
    - `acp_sofdsp_dai_links_create` creates `snd_soc_dai_link`s
    - `devm_snd_soc_register_card` registers a card
