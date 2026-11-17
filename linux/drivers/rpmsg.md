# Linux rpmsg

## Device

- an `rpmsg_device` is a virtual device that represents a channel to a remote
  processor
- when used with rproc,
  - `rproc_add_subdev` adds a subdev
  - `rproc_start` calls `rproc_start_subdevices`
  - subdev `start` callback calls `rpmsg_register_device`
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

## `CONFIG_RPMSG_QCOM_GLINK_SMEM`

- some qcom remote processors use glink-over-smem to communicate with the host
  - the rproc dt node has a `glink-edge` child node
  - the rproc driver calls `qcom_add_glink_subdev` to parse `glink-edge` child
    - it adds the glink child node as a subdev
  - `rproc_start` starts the rproc and calls `rproc_start_subdevices`
    - `glink_subdev_start` calls `qcom_glink_smem_register`
- `qcom_glink_smem_register` registers a `glink-edge` child node of a rproc
  - there appears to be two ring buffers, tx and rx, in `qcom,smem`
    - `CONFIG_QCOM_SMEM` enables `qcom,smem` support
    - on tx, `smem->mbox_chan` notifies rproc via mailbox
    - on rx, `qcom_glink_smem_intr` notifies host via irq
  - `qcom_glink_native_probe` starts the handshake with rproc
- glink handshake and rpmsg dev
  - `qcom_glink_send_version` sends `GLINK_CMD_VERSION`
  - `qcom_glink_receive_version_ack` handles `GLINK_CMD_VERSION_ACK`
  - `qcom_glink_receive_version` handles `GLINK_CMD_VERSION`
  - `qcom_glink_send_version_ack` sends `GLINK_CMD_VERSION_ACK`
  - `qcom_glink_rx_open` handles `GLINK_CMD_OPEN`
    - `qcom_glink_alloc_channel` allocs a channel
    - `rpmsg_register_device` registers an rpmsg dev for the channel
      - the name is specified by rproc, such as `PMIC_RTR_ADSP_APPS`
      - `glink_device_ops` is the ops
- when an rpmsg drv probes the rpmsg dev,
  - bus `rpmsg_dev_probe` calls `rpmsg_create_ept -> qcom_glink_create_ept` to
    create ept
  - `qcom_glink_create_remote` continues the handshake
    - `qcom_glink_send_open_ack` sends `GLINK_CMD_OPEN_ACK`
    - `qcom_glink_send_open_req` sends `GLINK_CMD_OPEN`
    - `qcom_glink_rx_open_ack` handles `GLINK_CMD_OPEN_ACK`
  - there is also `qcom_glink_create_local` which is mainly used when driven
    from userspace via `rpmsg_ctrl` created in `qcom_glink_create_chrdev`
