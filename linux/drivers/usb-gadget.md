USB Gadget
==========

## `CONFIG_USB_GADGET`

- there are hw drivers for UDCs (USB Device/Peripheral Controller)
  - we typically don't need these because we rely on DRD or OTG
- there are legacy sw drivers for simulated functions
  - `CONFIG_USB_G_SERIAL` simulates CDC ACM
    - it talks CDC ACM protocol to the UDC
    - it creates `/dev/ttyGS0` for userspace to provide tx/rx data
    - the host side needs `CONFIG_USB_ACM` to drive this gadget
    - internally, it uses
      - `f_acm.c` function to talk CDC ACM protocol
      - `u_serial.c` utility to interface tty subsystem to provide
        `/dev/ttyGS0`
- there are modern sw drivers for simulated functions
  - they are configured via configfs
  - `CONFIG_USB_CONFIGFS_ACM` simulates CDC ACM
    - `CONFIG_USB_F_ACM` compiles in `f_acm.c`
    - `CONFIG_USB_U_SERIAL` compiles in `u_serial.c`

## `CONFIG_USB_CONFIGFS`

- <https://docs.kernel.org/usb/gadget_configfs.html>
- `mount -t configfs none /config`
- to create a gadget,
  - `mkdir /config/usb_gadget/g1`
  - `cd /config/usb_gadget/g1`
  - `echo $VID > idVendor`
  - `echo $PID > idProduct`
  - `mkdir strings/0x409`
    - 0x409 stands for en-US
  - `echo $SERIAL > strings/0x409/serialnumber`
  - `echo $MANUFACTURER > strings/0x409/manufacturer`
  - `echo $PRODUCT > strings/0x409/product`
- to create a configuration for a gadget,
  - `mkdir configs/c.1`
  - `mkdir configs/c.1/strings/0x409`
  - `echo $CONFIGURATION > configs/c.1/strings/0x409/configuration`
- to create a function for a gadget,
  - `mkdir functions/acm.foo`
    - this loads `usb_f_acm.ko` automatically
- to associate a function with a configuration for a gadget,
  - `ln -s functions/acm.foo configs/c.1`
- to enable/disable a gadget,
  - `echo $UDC > UDC`
    - `$UDC` is `/sys/class/udc/$UDC`
  - `echo "" > UDC`
- to destroy a gadget,
  - `rm configs/c.1/acm.foo`
  - `rmdir functions/acm.foo`
  - `rmdir configs/c.1/strings/0x409`
  - `rmdir configs/c.1`
  - `rmdir strings/0x409`
  - `cd ..`
  - `rmdir g1`

## RK3588

- the soc has
  - xHCI 3.0 controllers x2
    - `usb_host0_xhci: usb@fc000000`
    - `usb_host2_xhci: usb@fcd00000`
  - EHCI 2.0 controllers x2
    - `usb_host0_ehci: usb@fc800000`
    - `usb_host1_ehci: usb@fc880000`
  - OHCI 1.0 controllers x2
    - `usb_host0_ohci: usb@fc840000`
    - `usb_host1_ohci: usb@fc8c0000`
- the device has
  - USB-A 2.0 ports x2
    - `u2phy2_host: host-port`
    - `u2phy3_host: host-port`
    - EHCI 2.0 and OHCI 1.0 controllers are connected to the phys
      - which controller to use depends on whether the connected device is 1.0
        or 2.0
  - USB-A 3.0 ports x1
    - `combphy2_psu: phy@fee20000`
    - `usb_host2_xhci: usb@fcd00000` controller is connected to the phy
  - USB-C ports x2
    - one of them is power-only and is not described by dt
    - `usb_con: connector` is the type-c connector
    - `usbc0: usb-typec@22` is the type-c controller connected to the connector
    - `usbdp_phy0` is DP altmode phy connected to the type-c controller
    - `u2phy0_otg: otg-port` is USB phy connected to the type-c controller
    - `usb_host0_xhci: usb@fc000000` is the xhci controller connected to the USB phy
