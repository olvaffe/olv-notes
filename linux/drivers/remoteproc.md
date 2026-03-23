Kernel remoteproc
=================

## Overview

- remoteproc is a framework to control a remote processor
  - to power on
  - to power off
  - to load fw
- rproc does not provide a bus to match a driver to a device
- an rproc driver discovers its remote processor through other means and...
  - `devm_rproc_alloc` creates a `rproc`
    - this specifies `rproc_ops` and the fw name
  - `rproc_add_subdev` optionally adds a subdev to rproc
    - these subdevices are started/stopped with the rproc
  - `rproc_add` adds the rproc to the core
    - if `rproc->auto_boot` is set, `rproc_boot` is called automatically
  - `rproc_boot` boots a rproc
    - `request_firmware` loads the fw to memory
    - `rproc_fw_boot` powers on rproc to run the fw
      - `rproc_start` starts the proc and calls `rproc_start_subdevices` to
        start subdevs
  - `rproc_shutdown` powers off a rproc
- the fw seems to be standardized somewhat
  - must be in ELF32 or ELF64
- the fw can support `RSC_VDEV`, which uses virtio as a transport
  - it is not about virtualization anymore
  - `rproc_handle_vdev` creates `rproc-virtio` pdev with fw-providied id,
    typically `VIRTIO_ID_RPMSG`
  - `rproc_virtio_probe` probes pdev, allocs vrings, and calls
    `rproc_add_subdev` to add a subdev
  - when it boots, `rproc_vdev_do_start` calls `rproc_add_virtio_dev` to
    register a `virtio_device`
  - `CONFIG_RPMSG_VIRTIO` is the virtio driver for the virtio device
    - `rpmsg_probe` calls `rpmsg_ctrldev_register_device` and optionally
      `rpmsg_ns_register_device` to register `rpmsg_device`

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
