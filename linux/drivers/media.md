Linux Media Subsystem
=====================

## Overview

- a driver can normally be found under
  - `pci` for PCI devices
  - `usb` for USB devices
  - `platform` for platform devices
  - some ir or radio devices are under `rc` or `radio` etc.
- DVB devices
  - the device usually has a I2C bus with demodulator and tuner
    - call `i2c_new_client_device` to statically add the devices
  - `dvb_register_adapter`
  - `dvb_register_frontend`
  - `dvb_dmxdev_init`
  - `dvb_net_init`

## Video4Linux Devices

- `media-v4l.md`

## Digital TV (DVB) Devices

- overview
  - antenna passes raw signal to the frontend
  - the frontend downconverts and demodulates the signal into an MPEG
    transport stream (TS)
    - in case of satellite frontend, there is satellite equipment control
      (SEC) that allows control of multi feed switches, dish rotors, etc.
  - the conditional access (CA) decodes the programs the user has access to
  - the demultiplexer splits the TS into video, audio, and data streams
  - video and audio decoders (modern hardware do not have them, but rely on
    CPU/GPU)
- there are
  - `/dev/dvb/adapterY/frontendX`
  - `/dev/dvb/adapterY/caX`
  - `/dev/dvb/adapterY/demuxX`
  - `/dev/dvb/adapterY/videoX`
  - `/dev/dvb/adapterY/audioX`
  - `/dev/dvb/adapterY/netX`
    - ip-over-satellite
    - create virtual network interface for the demuxed data stream
  - `/dev/dvb/adapterY/dvrX`
    - digital video recorder

## Remote Controller Devices

- most media devices have infrared receivers
- the IR receiver receives a series of pulses and pauses.  The IR decoder
  decodes it to a vendor-defined scancode.  A keymap is used to map the
  scancode to linux keycodes.
  - `ir_raw_event_store` adds a raw event to a FIFO
  - `ir_raw_event_handle` wakes up a kthread to pass raw events to decoders
  - `ir_raw_handler` can
    - encode a scancode into a series of `ir_raw_event`
    - decode a series of `ir_raw_event` and report the scancode via
      `rc_keydown`
- `/dev/lircX` and ioctls
  - bidirectional

## Media Controller Devices

- `/dev/mediaX`
- an advanced media device has many seemingly unrelated devices, such as
  video capture, microphone, video output, video encode, etc.
- some of the devices have dependencies on each other
- MC api provides userspace a way to query and configure device topology 

## Consumer Electronics Control Devices

- `/dev/cecX` and ioctls
- CEC pin of HDMI connector
- `CEC_CAP_RC` means the adaptor supports remote control protocol
