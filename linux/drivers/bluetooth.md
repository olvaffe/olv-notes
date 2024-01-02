Linux bluetooth driver
======================

## HCI over UART

- `btattach` opens the serial port
  - Makes it raw and sets the baud rate
  - Changes ldisc and proto
    - `TIOCSETD` to use `N_HCI` ldisc.
    - `HCIUARTSETPROTO` to specify protocol (H4, BCSP, LL, etc.)
- The `N_HCI` ldisc
  - does not allow read/write from the tty.
- When a proto is set through `HCIUARTSETPROTO`
  - the proto is `open`ed.
  - `hci_uart_register_dev` is called to register a hci device.
- When there are outgoing data, `hci_send_frame` is called
  - In uart case, `hci_uart_send_frame` is called.
  - `skb` is always enqueued to proto first.
  - They are dequeued from proto and written to the serial port.
- When there are incoming data, ldisc `receive_buf` is called.
  - It calls proto's `recv`, like `ll_recv`.
  - The buffer is parsed to form `skb`s.
  - `hci_recv_frame` is called to receive the `skb`s.
