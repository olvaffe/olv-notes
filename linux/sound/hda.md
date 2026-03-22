Linux hda
=========

## Overview

- Intel High Definition Audio, developed by intel and codec suppliers in 2004
  - host controller is on the pci bus and bridges cpu and codecs
    - suppliers include intel, amd, nvidia, via, creative, etc.
  - codecs sit behind the host controller
    - suppliers include realtek, etc
  - `CPU <-> Controller <-> Codecs <-> Inputs/Outputs`
- hda is a bus and a codec is a device on the bus
- `sounds/hda`
  - `codecs` contains codec drivers
  - `common` provides `snd-hda-codec`
    - controller drivers use
      - `azx_bus_init` to create an hda bus
      - `azx_probe_codecs` to probe codecs
      - etc.
    - codec drivers use
      - `module_hda_codec_driver` to register a codec driver
      - etc.
  - `controllers` are controller drivers
    - intel no longer uses `snd-hda-intel`
    - amd uses `snd-hda-intel` for output but not input
  - `core` provides `snd-hda-core` and `snd-hda-ext-core`
    - it registers hda bus type, provides low-level hw communications, etc.
- intel dsp with sof
  - there is a dsp driver, such as `CONFIG_SND_SOC_SOF_PANTHERLAKE`
  - if dsp supports hda, `CONFIG_SND_SOC_SOF_HDA_LINK` is like the hda
    controller driver
    - this is enough for gpu hdmi audio, which needs `core` but not `common`
  - if there are hda codecs, `CONFIG_SND_SOC_SOF_HDA_AUDIO_CODEC` is like the
    hda codec driver
    - it selects `CONFIG_SND_SOC_HDAC_HDA` to adapt hda codec drivers for
      asoc, which needs both `core` and `common`

## Legacy HD Audio

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

## AMD HDMI

- gpu pci device has two functions
  - `0e:00.0 VGA compatible controller ...`
  - `0e:00.1 Audio device ...`
- amdgpu probes the first function and calls `amdgpu_dm_audio_init`
  - `component_add` adds a component with `amdgpu_dm_audio_component_bind_ops`
- `azx_probe` probes the second function
  - `azx_probe_codecs` finds a codec with id `0x1002aa01`
  - `azx_codec_configure` registers the codec to the hda bus
- `atihdmi_probe` probes the hdmi codec
  - `snd_hda_hdmi_acomp_init -> snd_hdac_acomp_init`
    - `component_match_add_typed` adds a match with `match_bound_vga`
    - `component_master_add_with_match` adds a master with
      `hdac_component_master_bind`
- `hdac_component_master_bind` binds the component after all 3 drivers probe
  - `component_bind_all` calls `amdgpu_dm_audio_component_bind`
    - it makes `amdgpu_dm_audio_component_ops` available to hda

## Intel HDMI

- on an old broadwell, there are two hdaudio pci devices
  - `8086:160c` has `AZX_DRIVER_HDMI` and `AZX_DCAPS_INTEL_BROADWELL`
  - `8086:9ca0` has `AZX_DRIVER_PCH` and `AZX_DCAPS_INTEL_PCH`
  - the legacy azx driver probes both pci devices
- i915 `intel_display_driver_register`
  - `intel_audio_init` inits audio clk
  - `intel_audio_register` calls `component_add_typed` to add a
    `I915_COMPONENT_AUDIO` component with `intel_audio_component_bind_ops`
- for `8086:160c`, `azx_probe` calls `snd_hdac_i915_init` which calls
  `snd_hdac_acomp_init`
  - `component_match_add_typed` adds a component match using
    `i915_component_master_match` to match the component added by i915
  - `component_master_add_with_match` binds to the component added by i915
  - `hdac_component_master_bind` calls `intel_audio_component_bind`
  - the audio driver now has access to to `intel_audio_component_ops`
- on `8086:160c` hda bus, there is a codec of id `0x80862808`
  - `module_hda_codec_driver(hdmi_driver)` registers the codec driver
  - `hda_codec_driver_probe` probes the codec
  - `probe_i915_hsw_hdmi` probes the codec
  - there are 3 audio output nodes
- on `8086:9ca0` hda bus, there is a codec of id `0x10ec0288`
  - `module_hda_codec_driver(realtek_driver)` registers the codec driver
  - `hda_codec_driver_probe` probes the codec
  - `alc269_probe` probes the codec
  - there are 3 audio output nodes and 3 audio input nodes

## `snd_intel_dsp_driver_probe`

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
