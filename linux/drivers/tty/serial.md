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
