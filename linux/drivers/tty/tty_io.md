Kernel TTY IO
=============

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

## Init

- `chr_dev_init` calls `tty_init` to register chrdevs
  - `/dev/tty` has `MKDEV(TTYAUX_MAJOR, 0)`
  - `/dev/console` has `MKDEV(TTYAUX_MAJOR, 1)`
  - if vt is enabled, `vty_init` is called
    - `/dev/tty0` has `MKDEV(TTY_MAJOR, 0)`
- `/dev/tty` uses `tty_fops`
  - on open, `tty_open` calls `tty_open_current_tty`
    - this returns `current->signal->tty`, the controlling tty
- `/dev/console` and `/dev/tty0` both use `console_fops`
  - on open, `tty_open` calls `tty_open_by_driver`
    - `tty_lookup_driver` returns the `tty_driver` and the dev idx
      - for `/dev/console`, this calls `console_device` from printk
        - it returns the first tty console in `console_list`
        - the last `console=` specifies the preferred console and
          `register_console` keeps it at the head of `console_list`
      - for `/dev/tty0`, this returns `console_driver` and `fg_console` from
        vt

## `tty_driver`

- other subsystems may call `tty_register_driver` to register tty drivers
  - bt rfcomm registers `rfcomm_tty_driver` for `/dev/rfcomm*`
  - usb serial registers `usb_serial_tty_driver` for `/dev/ttyUSB*`
  - usb cdc-acm registers `acm_tty_driver` for `/dev/ttyACM*`
  - usb cdc-acm registers `acm_tty_driver` for `/dev/ttyACM*`
  - sdio uart registers `tty_drv` for `/dev/ttySDIO*`
- tty subsystem has these `tty_driver`
  - pty registers `ptm_driver` and `pts_driver`
    - legacy pty registers `pty_driver` and `pty_slave_driver`
  - rpmsg registers `rpmsg_tty_driver` for `/dev/ttyRPMSG*`
  - serial registers `normal`
    - `/dev/ttyS*` for 8250 and several others
    - `/dev/ttyAMA*` for AMBA PL011
    - `/dev/ttyMSM*` for qcom MSM and GENI
    - `/dev/ttyHS*` for qcom GENI
  - ttynull registers for `/dev/ttynull`
  - vt registers `console_driver` for `/dev/tty*`
- each driver has a `tty_operations`
  - tty fops forwards many ops to `tty_operations`

## `file_operations`

- the tty subsystem has these `file_operations`
  - `tty_fops`
  - `console_fops` is only used by `/dev/console` and `/dev/tty0`
    - it is the same as `tty_fops` except it supports `TIOCCONS` redirect
  - `hung_up_tty_fops` is only used after a tty is hung up
    - it stubs out all ops
- `tty_fops::open` is `tty_open`
  - if rdev is `MKDEV(TTYAUX_MAJOR, 0)`, that is `/dev/tty`,
    `tty_open_current_tty` returns the controlling tty
  - otherwise, `tty_open_by_driver` opens the tty
    - if rdev is `MKDEV(TTY_MAJOR, 0)`, that is `/dev/tty0`, it uses vt
      `console_driver`
    - if rdev is `MKDEV(TTYAUX_MAJOR, 1)`, that is `/dev/console`, it uses
      printk `console_device`
    - otherwise, `get_tty_driver` searches `tty_drivers` for drivers
      registered with `tty_register_driver`
  - `tty_init_dev` calls `tty_ldisc_init` indirectly and uses `N_TTY` line
    discipline by default
- `tty_fops::read_iter` is `tty_read` and reads via ldisc
  - tty drivers typically call `tty_flip_buffer_push` to push data to ldisc
    for reading
- `tty_fops::write_iter` is `tty_write` and writes via ldisc
- `tty_fops::unlocked_ioctl` is `tty_ioctl`
  - it handles a bunch of `TIOC*` ioctls
  - `tty_jobctrl_ioctl` handles more `TIOC*` ioctls related to job control
  - the tty driver handles more ioctls
    - serial handles some `TIOC*` ioctls
    - vt handles `KD*` and `VT_*` ioctls
  - the ldisc handles `TIOC*` and `TC*` ioctls
