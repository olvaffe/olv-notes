Kernel remoteproc
=================

## Overview

- remoteproc is a framework to control a remote processor
  - to power on
  - to power off
  - to load fw
  - nothing more
- the fw seems to be standardized somewhat
  - must be in ELF32 or ELF64
- a remote processor "driver"
  - `devm_rproc_alloc` to create a `rproc`
    - this specifies `rproc_ops` and the fw name
  - `rproc_add_subdev` to optionally add a subdev to rproc
    - these subdevices are started/stopped with the rproc
  - `rproc_add` to add the rproc

## Consumer

- `rproc_boot` boots a rproc
  - `request_firmware` loads the fw to memory
  - `rproc_fw_boot` powers on rproc to run the fw
- `rproc_shutdown` powers off a rproc
