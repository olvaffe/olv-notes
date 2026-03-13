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
- configs
  - `CONFIG_RC_DEVICES` are drivers for ir receiver/transmitter
  - `CONFIG_RC_DECODERS` are raw event handlers to encode/decode between raw
    events (ir pulses and pauses) and scancodes
  - `CONFIG_RC_MAP` are key maps to map scancodes to keycodes
- the IR receiver receives a series of pulses and pauses.  The IR decoder
  decodes it to a vendor-defined scancode.  A keymap is used to map the
  scancode to linux keycodes.
  - `ir_raw_event_store` adds a raw event to a FIFO
  - `ir_raw_event_handle` wakes up `ir_raw_event_thread`
    - it forwards raw events to raw handlers for decoding
    - it also calls `lirc_raw_event` to forward raw events to lirc interface
- `ir_raw_handler`
  - `ir_raw_handler_register` registers a raw handler
  - it can decode a series of `ir_raw_event` and report the scancode via
    `rc_keydown`
  - it can encode a scancode into a series of `ir_raw_event` for sending
- `rc_keydown`
  - `rc_g_keycode_from_table` maps scancode to keycode
    - `rc_map_register` registers a map
    - when `rc_register_device` registers the receiver, `rc_map_get` returns
      the key map for the receiver
  - `lirc_scancode_event` sends the scancode and keycode to userspace via lirc
    interface
  - `input_*` sends the scancode and keycode to input subsystem
- `/dev/lircX` and ioctls
  - since rc subsys also reports scancode/keycode via input subsys, this
    interface is usually not needed
  - but when the kernel does not know how to decode the raw events or map the
    scancode,
    - lircd can use the interface to do the decoding and mapping in userspace
    - it is also possible to attach a bpf program via the interface to do the
      decoding and mapping
      - `lirc_raw_event` calls `lirc_bpf_run` to run the bpf program
      - `bpf_rc_keydown` calls `rc_keydown`

## Media Controller Devices

- `/dev/mediaX`
- an advanced media device has many seemingly unrelated devices, such as
  video capture, microphone, video output, video encode, etc.
- some of the devices have dependencies on each other
- MC api provides userspace a way to query and configure device topology 

## Consumer Electronics Control Devices

- CEC pin of HDMI connector
- init
  - `cec_allocate_adapter` allocates an adapter
    - if `CEC_CAP_RC`, `rc_allocate_device` also allocates an rc device
      - the cap means the adaptor supports remote control protocol
  - `cec_register_adapter` registers an adapter
    - if `CEC_CAP_RC`, `rc_register_device` also registers the rc device
- upon receiving an msg, driver calls `cec_received_msg`
  - `cec_receive_notify` processes the msg
    - if `CEC_MSG_USER_CONTROL_PRESSED`, it calls `rc_keydown` to forward the
      scancode to rc subsys
      - no raw event or raw handler needed
      - `RC_MAP_CEC` maps cec scancode to keycode
- on desktop gpus, they rely on dp aux to tunnel cec
  - `drm_dp_cec_register_connector`
  - amdgpu/i915/nouveau calls `drm_dp_cec_attach` directly or indirectly to
    allocate the cec adapter
- `/dev/cecX` and ioctls
