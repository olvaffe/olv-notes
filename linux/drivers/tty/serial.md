Kernel TTY Serial
=================

## HW

- parallel interface
  - e.g., 8-bit data bus plus a clock require 9 wires
- serial interface
  - e.g., 2 wires for data and clock respectively
- synchronous serial
  - e.g., spi, i2c, etc, which requires a wire for clock
- asynchronous serial
  - just 1 data wire, no clock wire
  - two devices must be configured to use exactly the same protocol
- protocol
  - baud rate, in bits-per-second
    - it determines how long a transmitter holds the serial line high/low and
      what period the receiver samples the serial line
  - start bit
  - data bits
  - parity bit
  - stop bit(s)
  - e.g., `9600 8N1` means
    - baud rate is 9600
    - start bit is always 1 bit
    - data bits are 8
    - no parity bit
    - stop bit is 1 bit
- UART stands for universal asynchronous receiver/transmitter

## earlycon

- `earlycon` is an early param parsed by `parse_early_param`
  - if a value is specified, `setup_earlycon` parses value and calls
    `register_earlycon`
  - if no value is specified, `early_init_dt_scan_chosen_stdout` parses
    `stdout-path` from `/chosen` and calls `of_setup_earlycon`
- `OF_EARLYCON_DECLARE` declares an earlycon driver
  - all drivers are collected to `__earlycon_table` and `__earlycon_table_end`
- regular console and earlycon
  - `start_kernel`
    - `setup_arch` is called early
      - `parse_early_param` enables earlycon
    - `rest_init` is called last
      - `user_mode_thread` spawns pid 1 to run `kernel_init`
  - `kernel_init`
    - `kernel_init_freeable`
      - `do_basic_setup`
        - `do_initcalls` enables regular console
          - it uses `device_initcall` so it is pretty late
