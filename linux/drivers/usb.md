USB
===

## lspci / lsusb

- Lenovo X1 Gen9
  - specs
    - 1x Thunderbolt 4 power input
    - 1x Thunderbolt 4
    - 2x USB-A 3.2 Gen1
  - `lsusb -t`
    - 2x 4-port USB 3.2 Gen2 (10Gbit/s) root hubs
      - not used; used only when TB or USB3 device is plugged in?
    - 1x 1-port USB 2.0 (480Mbit/s) root hub
      - not used
    - 1x 12-port USB 2.0 (480Mbit/s) root hub
      - touchpad (12Mbit)
      - camera (480Mbit)
      - bluetooth (12Mbit)
      - all 4 USB Type-A and Type-C ports
  - I connect these to the ports
    - security key (12Mbit)
    - usb dock (480Mbit)
      - built-in usb ethernet (480Mbit)
      - monitor (480Mbit)
        - keyboard (12Mbit) and mouse (12Mbit)
- lspci -v
  - one xHCI USB controller driven by `xhci_hcd` pci driver
- most USB devices are driven by a `usb_device_driver` called `usb`
  - the driver enumerates usb interfaces that can be driven by `usb_driver`s
- A USB hub, root or not, is driven by a `usb_driver` called `hub`
- A root hub starts a new bus
