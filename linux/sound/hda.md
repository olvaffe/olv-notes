Linux hda
=========

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
    - `azx_driver` is a driver for the traditional pci device and the pci
      device is a controller for an hda bus
    - other devices can have controllers for hda buses as well
      - they also call `snd_hdac_bus_init*` to init an hda bus

## Intel

- `snd_intel_dsp_driver_probe`
  - it checks vendor id, device id, device class, etc. to see if the pci
    deivce has a DSP
  - if the pci device has a dsp, it uses `config_table` to determine how to
    drive the device
  - `SND_INTEL_DSP_DRIVER_LEGACY` uses `CONFIG_SND_HDA_INTEL`, the legacy azx
    (`snd_hda_intel`) driver
  - `SND_INTEL_DSP_DRIVER_SST` uses one of
    - `CONFIG_SND_SST_ATOM_HIFI2_PLATFORM_ACPI`
    - `CONFIG_SND_SOC_INTEL_CATPT`
    - `CONFIG_SND_SOC_INTEL_AVS`
  - `SND_INTEL_DSP_DRIVER_AVS` uses `CONFIG_SND_SOC_INTEL_AVS`
  - `SND_INTEL_DSP_DRIVER_SOF` uses `CONFIG_SND_SOC_SOF_HDA_GENERIC`
- dmesg on tgl

    snd_hda_intel 0000:00:1f.3: DSP detected with PCI class/subclass/prog-if info 0x040380
    snd_hda_intel 0000:00:1f.3: Digital mics found on Skylake+ platform, using SOF driver
    snd_soc_avs 0000:00:1f.3: DSP detected with PCI class/subclass/prog-if info 0x040380
    snd_soc_avs 0000:00:1f.3: Digital mics found on Skylake+ platform, using SOF driver
    sof-audio-pci-intel-tgl 0000:00:1f.3: DSP detected with PCI class/subclass/prog-if info 0x040380
    sof-audio-pci-intel-tgl 0000:00:1f.3: Digital mics found on Skylake+ platform, using SOF driver
    sof-audio-pci-intel-tgl 0000:00:1f.3: DSP detected with PCI class/subclass/prog-if 0x040380
    sof-audio-pci-intel-tgl 0000:00:1f.3: bound 0000:00:02.0 (ops i915_audio_component_bind_ops [i915])
    sof-audio-pci-intel-tgl 0000:00:1f.3: use msi interrupt mode
    sof-audio-pci-intel-tgl 0000:00:1f.3: hda codecs found, mask 5
    sof-audio-pci-intel-tgl 0000:00:1f.3: using HDA machine driver skl_hda_dsp_generic now
    sof-audio-pci-intel-tgl 0000:00:1f.3: DMICs detected in NHLT tables: 4
    sof-audio-pci-intel-tgl 0000:00:1f.3: Firmware paths/files for ipc type 0:
    sof-audio-pci-intel-tgl 0000:00:1f.3:  Firmware file:     intel/sof/sof-tgl.ri
    sof-audio-pci-intel-tgl 0000:00:1f.3:  Topology file:     intel/sof-tplg/sof-hda-generic-4ch.tplg
    sof-audio-pci-intel-tgl 0000:00:1f.3: Firmware info: version 2:2:0-57864
    sof-audio-pci-intel-tgl 0000:00:1f.3: Firmware: ABI 3:22:1 Kernel ABI 3:23:0
    sof-audio-pci-intel-tgl 0000:00:1f.3: unknown sof_ext_man header type 3 size 0x30
    sof-audio-pci-intel-tgl 0000:00:1f.3: Firmware info: version 2:2:0-57864
    sof-audio-pci-intel-tgl 0000:00:1f.3: Firmware: ABI 3:22:1 Kernel ABI 3:23:0
    sof-audio-pci-intel-tgl 0000:00:1f.3: Topology: ABI 3:22:1 Kernel ABI 3:23:0
    skl_hda_dsp_generic skl_hda_dsp_generic: ASoC: Parent card not yet available, widget card binding deferred
    snd_hda_codec_realtek ehdaudio0D0: autoconfig for ALC287: line_outs=2 (0x14/0x17/0x0/0x0/0x0) type:speaker
    snd_hda_codec_realtek ehdaudio0D0:    speaker_outs=0 (0x0/0x0/0x0/0x0/0x0)
    snd_hda_codec_realtek ehdaudio0D0:    hp_outs=1 (0x21/0x0/0x0/0x0/0x0)
    snd_hda_codec_realtek ehdaudio0D0:    mono: mono_out=0x0
    snd_hda_codec_realtek ehdaudio0D0:    inputs:
    snd_hda_codec_realtek ehdaudio0D0:      Mic=0x19
    skl_hda_dsp_generic skl_hda_dsp_generic: hda_dsp_hdmi_build_controls: no PCM in topology for HDMI converter 3
    input: sof-hda-dsp Mic as /devices/pci0000:00/0000:00:1f.3/skl_hda_dsp_generic/sound/card0/input19
    input: sof-hda-dsp Headphone as /devices/pci0000:00/0000:00:1f.3/skl_hda_dsp_generic/sound/card0/input20
    input: sof-hda-dsp HDMI/DP,pcm=3 as /devices/pci0000:00/0000:00:1f.3/skl_hda_dsp_generic/sound/card0/input21
    input: sof-hda-dsp HDMI/DP,pcm=4 as /devices/pci0000:00/0000:00:1f.3/skl_hda_dsp_generic/sound/card0/input22
    input: sof-hda-dsp HDMI/DP,pcm=5 as /devices/pci0000:00/0000:00:1f.3/skl_hda_dsp_generic/sound/card0/input23
