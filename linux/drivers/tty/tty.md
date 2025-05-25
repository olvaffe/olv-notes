TTY
===

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
