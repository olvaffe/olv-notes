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
    - `usb_host0_xhci: usb@fc000000` paired with `u2phy0_otg: otg-port` and `usbdp_phy0: phy@fed80000`
    - `usb_host2_xhci: usb@fcd00000` paired with `combphy2_psu: phy@fee20000`
  - EHCI 2.0 controllers x2
    - `usb_host0_ehci: usb@fc800000` paired with `u2phy2_host: host-port`
    - `usb_host1_ehci: usb@fc880000` paired with `u2phy3_host: host-port`
  - OHCI 1.0 controllers x2
    - `usb_host0_ohci: usb@fc840000` paired with `u2phy2_host: host-port`
    - `usb_host1_ohci: usb@fc8c0000` paired with `u2phy3_host: host-port`
- orange pi 5 has
  - USB-A 3.0 port
    - `usb_host2_xhci: usb@fcd00000` and `combphy2_psu: phy@fee20000` if usb3
    - `usb_host1_ehci: usb@fc880000` and `u2phy3_host: host-port` if usb2
    - `usb_host1_ohci: usb@fc8c0000` and `u2phy3_host: host-port` if usb1
  - USB-A 2.0 port (below USB-A 3.0 port)
    - `usb_host0_ehci: usb@fc800000` and `u2phy2_host: host-port` if usb2
    - `usb_host0_ohci: usb@fc840000` and `u2phy2_host: host-port` if usb1
  - USB-A 2.0 port (OTG default to device mode)
    - `usb_host0_xhci: usb@fc000000` and `u2phy0_otg: otg-port` for usb2/1
  - USB-C port (power only)
    - not described by dt
  - USB-C port
    - `usb_host0_xhci: usb@fc000000` and `usbdp_phy0: phy@fed80000` if usb3
    - `usb_host0_xhci: usb@fc000000` and `u2phy0_otg: otg-port` if usb2/1
      - shared with USB-A 2.0 port OTG
    - `usbc0: usb-typec@22` is the type-c controller connected to the connector
      - `usb_con: connector` is the type-c connector
- `usb_con: connector` has
  - `data-role = "dual";`
  - `power-role = "dual";`
  - `typec_get_fw_cap` parses them to
    - `TYPEC_PORT_DRD`
    - `TYPEC_PORT_DRP`
  - `/sys/class/typec/port0` has
    - `data_role` can be `host` or `device`
    - `port_type` can be `dual`, `source`, or `sink`
    - `power_role` can be `source` or `sink`
- `usbdp_phy0: phy@fed80000` has
  - `mode-switch;`
    - `rk_udphy_setup_typec_mux` calls `typec_mux_register`
    - this is how the phy supports altmode
  - `orientation-switch;`
    - `rk_udphy_setup_orien_switch` calls `typec_switch_register`
    - this is how the phy supports both orientations
- `usb_host0_xhci: usb@fc000000` has
  - `dr_mode = "otg";`
    - `dwc3_get_properties` calls `usb_get_dr_mode` to parse `otg` to
      `USB_DR_MODE_OTG`
  - `usb-role-switch;`
    - `dwc3_setup_role_switch` calls `usb_role_switch_register`
    - because it does not set `allow_userspace_control`, the role cannot be
      switched from userspace
      - have to use `/sys/kernel/debug/usb/fc000000.usb/mode` instead
