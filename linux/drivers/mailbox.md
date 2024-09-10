Kernel Mailbox
==============

## Controllers

- a mailbox is a mechanism for inter-processor communication
  - between the ap processor and various coprocessors
- controller driver
  - allocates a `mbox_controller`
  - allocates a `mbox_chan` for each channel
  - inits controller and channel structs
    - especially `mbox_chan_ops`
  - `devm_mbox_controller_register` to register

## Consumers

- consumer driver
  - `mbox_request_channel` to request a `mbox_chan`
  - `mbox_send_message` to send a request
    - the controller will forward the request to the coprocessor attached to
      the channel
    - when a response is available, `rx_callback` is called
