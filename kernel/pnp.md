PNP
===

## Core

- drivers/pnp/core.c registers the pnp bus, `pnp_bus_type`
- there are several PNP procotols (isa, bios, acpi) and they are registered
  with `pnp_register_protocol`
  - a protocol itself is a `struct device`, named pnpX
- to add a device,
  - `pnp_alloc_dev` allocates a new `struct pnp_dev` under pnpX
  - set up device caps and resources
  - `pnp_add_device` registers the device
- to add a driver,
  - `pnp_register_driver` adds a driver
  - the core provides a `system` driver that reserves resources
- sysfs
  - all devices on the bus will have `pnp_dev_groups` attribute groups
  - `resources`
  - `options`
  - `id`

## ACPI PNP

- registers `pnpacpi_protocol` protocol
- calls `acpi_get_devices` and add PNP devices to PNP core
- `pnpacpi_add_device` registers a `pnp_device`, with caps and resources
  queried from ACPI

## Common Devices and Drivers

- `drivers/rtc/rtc-cmos.c`
  - `rtc_cmos` pnp driver
  - calls `devm_rtc_allocate_device` to add a `rtc_device` under the pnp
    device
  - sysfs attrs such as `date`, `time`, and `since_epoch`
- `drivers/input/serio/i8042.c`
  - `i8042-x86ia64io.h` provides `i8042 kbd` and `i8042 aux` pnp drivers
  - `i8042_platform_init` registers the drivers.  Their probe functions finds
    the io ports and irq such that `i8042_{read,write}_*` can work
  - i8042 calls `platform_create_bundle` to create a platform device under the platform
    bus
  - i8042 calls `serio_register_port` for each port (kbd and aux).  Each port
    is a child device of the platform device, but is under the serio bus
  - the drivers for the serio devices are `atkbd` and `psmouse` serio drivers
- `drivers/parport/parport_pc.c`
  - `parport_pc` pnp driver
  - it also registers a platform driver and a pci driver
  - `parport_register_port` registers a port, which is a device under parport
    bus
  - there are `parport_driver`s that call `parport_register_dev_model` to add
    the connected device
- `drivers/tty/serial/8250/8250_core.c`
  - calls `serial8250_pnp_init` to register the `serial` pnp driver
    - calls `serial8250_register_8250_port` which calls `uart_add_one_port` to
      register a 8250 port to serial core; serial core calls
      `tty_port_register_device_attr_serdev` to add a tty device
  - the core also adds a platform device, serial8250, and make sure there are
    `CONFIG_SERIAL_8250_RUNTIME_UARTS` ports by calling
    `serial8250_register_ports`.  By default, there are 4 ports so
    `serial8250_register_ports` adds (unusable) ttyS[1-3].
  - writing to ttySx goes through
    - file's `tty_fops.write`
    - ldisc's `n_tty_ops.write`
    - serial's `uart_ops.write`
    - when an unusable ttyS is opened, it goes through `tty_open`,
      `uart_open`, `tty_port_open`, `uart_port_activate`, `uart_startup`, and
      marks the tty `TTY_IO_ERROR`
