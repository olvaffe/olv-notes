# Kernel USB Serial

## Core

- each usb-serial driver uses `module_usb_serial_driver` to define a driver
  - it calls `usb_serial_register_drivers`
  - `usb_register` registers a usb driver
  - `usb_serial_register`
    - `usb_serial_operations_init` init default ops for the usb-serial driver
      - `usb_serial_generic_open`, `usb_serial_generic_write`, etc.
    - `usb_serial_bus_register` registers the usb-serial driver to the
      usb-serial bus
- when the usb driver probes, `usb_serial_probe`
  - creates a `usb_serial` corresponding to the usb device
  - creates a `usb_serial_port` for each usb device endpoint
  - adds each `usb_serial_port` to the usb-serial bus
- when the usb-serial driver probes, `usb_serial_device_probe`
  - this probes a `usb_serial_port`
  - `tty_port_register_device` registers the tty port with
    `usb_serial_tty_driver`

## USB Serial

- when usb urbs come in, usb serial driver parses the urbs
  - `tty_insert_flip_char` adds raw input char to `port->buf`
  - `tty_flip_buffer_push` queues `flush_to_ldisc`
  - `flush_to_ldisc` calls `n_tty_receive_buf` to receive raw input chars from
    `port->buf`, processes them, and add them to `tty->disc_data`
  - when userspace reads from the tty line, `tty_read` calls `n_tty_read` to
    read the data from `tty->disc_data`
- when userspace writes to the tty line, `tty_write` calls `n_tty_write`
  - it processes the raw data and calls `serial_write`
  - `usb_serial_generic_write`
    - `kfifo_in_locked` adds the processed data to `port->write_fifo`
    - driver-specific `prepare_write_buffer` copies the data from
      `port->write_fifo` to urb
    - `usb_submit_urb` submits the urb
