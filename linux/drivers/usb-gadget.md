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
