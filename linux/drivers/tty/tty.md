TTY
===

## Overview

- the application runs on the host with linux kernel
- a termial is connected to the host for I/O with a serial line
- the character set over the serial line is ASCII (7- or 8-bit)
- terminal as well as the serial line can be virtual
  - `pts` provides virtual lines
    - `gnome-terminal` uses `pts` to function as a terminal
    - An SSH connection uses `pts` on the remote (host)
  - linux VTs are virtual terminals and lines
- `termios` (or the linux-specific) `tty_ioctl` interface can be used to control
  the serial line on the host
- escape sequences are used to control the terminal from the host
  - Linux-specific(?) `console_ioctl` interface can be used to control the VT
- devices
  - `/dev/ttyS[0..3]` COM port serial line
  - `/dev/tty[0..64]` VT serial lines and terminals
  - `/dev/tty0` current VT
  - `/dev/tty` the controlling terminal
  - `/dev/console` kernel console (where kernel messages go to)

## Structs

- a `tty_driver` is allocated with `tty_alloc_driver`
  - after initializing the struct, the driver is registered with
    `tty_register_driver`
  - `num` specifies the number of lines; for each line, a `device` and a
    `cdev` are created.  The cdev fops is `tty_fops`
  - `tty_operations` is the operations of the driver
- a `tty_struct` is created when the device node is first opened
  - `tty_open` calls `tty_open_by_driver` to find the driver and return (or
    create) the `tty_struct`
  - if this is the first time, `tty_init_dev` creates the `tty_struct` with
    `ops` pointing to driver `ops` and `port` point to driver `ports[index]`.
    It then calls the install callback or `tty_standard_install` to add
    `tty_struct` to the driver
- a `tty_ldisc` sits between `file_operations` and `tty_operations`
  - for writes, it modifies the data and call `tty_operations::write`
  - for reads, it buffers and modifies data received (from `tty_port`).  It
    also writes (echos) the data back to the writer.
- a `tty_port` is for receiving data
  - when there is data to receive, normally notified by interrupts on HW,
    `tty_insert_flip_char` is called to add the data to `tty_buffer` of the
    port.  `tty_flip_buffer_push` or `tty_schedule_flip` is called on the
    `tty_port` to schedule `flush_to_ldisc`
  - `flush_to_ldisc` calls `tty_port_default_receive_buf` to flush the data to
    ldisc.

## PTY

- `/dev/ptmx` has `tty_fops` as its `file_operations` with `open` overriden
- when `/dev/ptmx` is opened, the driver checks if `/dev/pts` is mounted as
  devpts.  If so, it gets an index from devpts and
  `tty_init_dev(ptm_driver, index)` to initialize a master device.  It also
  calls `devpts_pty_new` to create a new inode in devpts.  `/dev/pts/<n>` uses
  the standard `tty_fops`
  - `tty_init_dev` calls `ptm_driver`'s `install`.  This sets up the slave
    `tty_struct`
  - it also sets up `tty_ldisc`
- when the master is written to, it goes through
  - `tty_write`
  - `n_tty_write`
  - `pty_write` adds the data to the input queue of the slave, and triggers
    `flush_to_ldisc` on the slave
  - `n_tty_receive_buf` on the slave is called
    - Among other things, it calls `echo_char`, `commit_echoes`, and
      `flush_echoes`.  Echos are written to the slave and are received by the
      master.
    - because `ptm_driver->init_termios` misses most flags, and ioctls against
      the master are redirected to the slave, the master ldisc is always in
      `real_raw` mode.
- when the slave is read, it goes through
  - `tty_read`
  - `n_tty_read`

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
