USB
===

## Hardware

- Speed
  - Low Speed: 1.5Mbit/s
  - Full Speed: 12Mbit/s
  - High Speed: 480Mbit/s, since 2.0
  - SuperSpeed: 5Gbit/s, since 3.0
    - also called 3.1 Gen1
  - SuperSpeed+: 10Gbit/s, since 3.1
    - also called 3.1 Gen2
  - USB 3.2 introduced
    - 3.2 Gen1x1: 5Gbit/s
    - 3.2 Gen1x2: 10Gbit/s (Gen1 with two-lane operation)
    - 3.2 Gen2x1: 10Gbit/s
    - 3.2 Gen2x2: 20Gbit/s (Gen2 with two-lane operation)
- Power
  - Low Power: 5V * 0.1A = 0.5W
  - High Power: up to 5V * 0.5A = 2.5W
  - Low Power SuperSpeed: 5V * 0.15A = 0.75W
  - High Power SuperSpeed: up to 5V * 0.9A = 4.5W
  - Two-Lane SuperSpeed (3.2 Gen2x2): up to 5V * 1.5A = 7.5W
  - BC 1.1: 5V * 1.5A = 7.5W
  - BC 1.2: 5V * 5A = 25W
  - USB-C: 5V * 1.5A = 7.5W
           5V * 3A = 15W
- Power Delivery
  - 1.0
    - 5V: 2A
    - 12V: 1.5A, 3A, 5A
    - 20V: 3A, 5A
  - 2.0/3.0
    - 5V: 0.1A ~ 3A
    - 9V: 1.67A ~ 3A
    - 15V: 1.8A ~ 3A
    - 20V: 2.25A ~ 5A
- Connectors
  - most common: type A, type C
  - was common: mini B, micro B
  - less common: type B
  - rare: mini A and micro A
- USB-C
  - Cable
    - SuperSpeed is optional
    - alternate mode is optional
    - PD is optional
- Chargers
  - MacBook: 87W
  - Laptop: 65W, 45W
  - Phone: 18W, 30W

## lspci / lsusb

- example on my laptop
  - two physical USB-A ports
- lspci -v
  - one xHCI USB controller driven by `xhci_hcd` pci driver
- lsusb -t
  - one USB 2.0 root hub
    - with built-in USB devices such as webcam, bluetooth, and touchscreen
      connected
  - one USB 3.0 root hub
    - with two USB-A ports for external USB devices
- most USB devices are driven by a `usb_device_driver` called `usb`
  - the driver enumerates usb interfaces that can be driven by `usb_driver`s
- A USB hub, root or not, is driven by a `usb_driver` called `hub`
- A root hub starts a new bus
