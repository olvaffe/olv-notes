# Linux rpmsg

## Device

- an `rpmsg_device` is a virtual device that represents a channel to a remote
  processor
- when used with rproc,
  - `rproc_add_subdev` adds a subdev
  - `rproc_start` calls `rproc_start_subdevices`
  - subdev `start` callback calls `rpmsg_register_device`
- e.g., some qcom remote processors use glink as the protocol
  - `glink_subdev_start` calls `qcom_glink_smem_register`
  - when `qcom_glink_smem_intr` receives `GLINK_CMD_OPEN`,
    `qcom_glink_rx_open` calls `rpmsg_register_device`
    - name is from fw, such as `PMIC_RTR_ADSP_APPS`
- e.g., virtio as the protocol
  - it is not about virtualization anymore
  - rproc creates a `rproc-virtio` pdev probed by `rproc_virtio_probe`
  - `rproc_vdev_do_start` registers a `virtio_device`
    - id is from hw, such as `VIRTIO_ID_RPMSG`
  - `rpmsg_probe` probes the virtio device
    - `rpmsg_virtio_add_ctrl_dev` calls `rpmsg_ctrldev_register_device`
  - `RPMSG_CREATE_DEV_IOCTL` calls `virtio_rpmsg_create_channel` which calls
    `rpmsg_register_device`
  - it looks like the main `rpmsg_device` is just a ctrl channel
    - `__rpmsg_create_channel` creates a sub-channel, which is another
      `rpmsg_device`

## Driver

- `module_rpmsg_driver` or `register_rpmsg_driver` registers a driver
- when bus finds a match, `rpmsg_dev_probe`
  - if the driver is simple and provides `rpdrv->callback`, the core calls
    `rpmsg_create_ept` for the driver
  - it calls driver probe
    - if the core hasn't, driver should call `rpmsg_create_ept` to create a
      `rpmsg_endpoint`
- communication
  - `rpmsg_send` sends a msg over ept
  - `rpmsg_rx_cb_t` is called for incomping msg from the ept
