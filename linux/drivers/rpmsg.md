Linux rpmsg
===========

## Driver

- `register_rpmsg_driver`
  - simple drivers usually just set `module_rpmsg_driver`
- when a `rpmsg_device` is matched, call `rpmsg_create_ept` to create a
  `rpmsg_endpoint`
  - this also have a `rpmsg_rx_cb_t` callback to handle responses
- `rpmsg_send` sends a packet over the endpoint

## Device

- when used with rproc, we can use `rproc_add_subdev` to add a subdev
  - when the rproc is booted, the subdev is prepared and we can use the
    callback to `rpmsg_register_device` a `rpmsg_device`
    - the `rpmsg_device` is itself a channel
    - it should have a `create_ept` callback to create an endpoint
  - this registers the device to be matched by a rpmsg driver
