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
