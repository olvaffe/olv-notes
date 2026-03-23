Kernel remoteproc
=================

## Overview

- remoteproc is a framework to control a remote processor
  - to power on
  - to power off
  - to load fw
- the fw seems to be standardized somewhat
  - must be in ELF32 or ELF64
- a remote processor "driver"
  - `devm_rproc_alloc` to create a `rproc`
    - this specifies `rproc_ops` and the fw name
  - `rproc_add_subdev` to optionally add a subdev to rproc
    - these subdevices are started/stopped with the rproc
  - `rproc_add` to add the rproc

- the fw can support `RSC_VDEV`, which uses virtio as a transport and is not
  about virtualization anymore
  - `rproc_handle_vdev` creates `rproc-virtio` pdev with `VIRTIO_ID_RPMSG`
  - `rproc_virtio_probe` probes and allocs vrings
  - when it boots, `rproc_vdev_do_start` calls `rproc_add_virtio_dev` to
    register a `virtio_device`
  - `CONFIG_RPMSG_VIRTIO` is the virtio driver for the virtio device

## Consumer

- `rproc_boot` boots a rproc
  - `request_firmware` loads the fw to memory
  - `rproc_fw_boot` powers on rproc to run the fw
- `rproc_shutdown` powers off a rproc

## `CONFIG_QCOM_Q6V5_PAS`

- `CONFIG_QCOM_Q6V5_PAS` is the adsp remoteproc driver
- it loads the adsp fw and boots adsp
- `qcom_pas_probe` calls `qcom_add_glink_subdev` to add a subdev
  - when it boots, `glink_subdev_start` calls `qcom_glink_smem_register`
    - `qcom_glink_native_probe`
      - `qcom_glink_rx_open` receives a channel named `PMIC_RTR_ADSP_APPS` and
        registers a rpmsg device of the name
- `CONFIG_QCOM_PMIC_GLINK` is the rpmsg driver for `PMIC_RTR_ADSP_APPS` rpmsg dev
  - its clients talk to adsp fw via the rpmsg dev
