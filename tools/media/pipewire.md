PipeWire
========

## Overview

- <https://gitlab.freedesktop.org/pipewire/pipewire.git>
- `pw-cli dump Core`
  - there is one `PipeWire:Interface:Core` object
- `pw-cli dump Module`
  - there are about a dozen `PipeWire:Interface:Module` objects
  - echo object is created by a `/usr/lib/pipewire-0.3/libpipewire-module-blah.so`
- `pw-cli dump Device`
  - there are several `PipeWire:Interface:Device` objects
  - some objects are backed by v4l2 devices
  - others are backed by alsa cards
- `pw-cli dump Node`
  - there are about a dozen `PipeWire:Interface:Node` objects
  - each object is a video/audio source/sink
  - there are also dummy ones
- `pw-cli dump Port`
  - there are about two dozens of `PipeWire:Interface:Port` objects
  - they are associated with `PipeWire:Interface:Node` objects
    - e.g., a speaker Node might have two Ports for left and right channels
- `pw-cli dump Factory`
  - there are about a dozen of `PipeWire:Interface:Factory` objects
  - echo object is created by a module
- `pw-cli dump Client`
  - there are a few `PipeWire:Interface:Client` objects
  - echo object is a current client that connects to pipewire
- `pw-cli dump Link`
  - there is no `PipeWire:Interface:Link` object unless something is actively
    playing
- `pw-cli dump Session`
  - there is no `PipeWire:Interface:Session` object
- `pw-cli dump Endpoint`
  - there is no `PipeWire:Interface:Endpoint` objects
- `pw-cli dump EndpointStream`
  - there is no `PipeWire:Interface:EndpointStream` objects

## API

- using `pw-cat` as an example
  - `pw_init` initializes the library
  - `pw_properties_new` and `pw_properties_set` create and manipulate
    `pw_properties`
  - `pw_main_loop_new` and `pw_main_loop_get_loop` create the main loop
  - `pw_context_new` creates `pw_context` to manage locally available
    resources
  - `pw_context_connect` creates `pw_core` and `pw_core_add_listener` listens
    to core events
  - `pw_stream_new` creates a `pw_stream`, `pw_stream_add_listener` listens
    to stream events, and `pw_stream_connect` connects a stream to data
    src/dst
  - `pw_main_loop_run`
- when `pw-cat` plays, it creates one Node, two Ports, and two Links
  - the Node has class `Stream/Output/Audio` and streams the music data
  - the two Ports are associated with the Node and are for the left/right
    channels
  - the two Links link the Ports of the Node to the Ports of a speaker Node

## Hotplug

- the alsa plugin, `libspa-alsa.so` supports hotplugs
  - spa stands for simple plugin api
- when the plugin is initialized, it calls `udev_monitor_new_from_netlink` to
  monitor udev events

## Session Manager

- I guess it listens to pipewire core events
- when a new Node shows up, it creates Links as needed
- with wireplumber
  - `wpctl get-volume @DEFAULT_AUDIO_SINK@`
  - `wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+`
