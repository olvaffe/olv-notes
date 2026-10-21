# Kernel USB Type-C

## Type-C HW

- type-c has 12 pins on each side
  - control plane: cc
  - power plane: vbus/gnd x2
  - data plane: tx1/rx1/tx2/rx2/sbu
  - 2.x compat: d+/d-
- control plane: `pdc/tcpc <-> port`
  - pdc runs fw
    - kernel talks to fw using ucsi or proprietary iface
    - fw is autonomous
  - tcpc is dumb
    - kernel talks to tcpc using tcpci or proprietary iface
    - kernel implements tcpm state machine
      - tcpc negotiates over cc pin
      - kernel updates soc, mux, retimer, etc.
- power plane: `src/sink <-> port`
  - control plane negotiates pd src/sink
  - as pd src, 5v rail connects to vbus
  - as pd sink, a charger ic connects to vbus
    - charge ic in turns outputs to vsys and battery
- data plane: `soc (usb3 and dp) <-> mux <-> retimer -> port`
  - soc can switch between host or device
- 2.x compat: `soc (usb2) <-> port`

## UCSI (USB Type-C Connector System Software Interface)

- UCSI is a standard interface that the os can use to communicate with PDC, if
  PDC FW supports the interface
  - PDC FW can work autonomously
  - kernel ucsi driver is optional; it is to get the port status or to
    override port power/data roles
- `CONFIG_TYPEC_UCSI` is a full UCSI implementation
- `CONFIG_UCSI_ACPI` is a driver for UCSI over ACPI
  - it registers a platform driver for `ucsi_acpi`
  - it calls `ucsi_create` and `ucsi_register`

## Type-C Port Controller Manager

- the os must provide a controller manager if the hw controller is not PDC but
  only dumb TCPC
  - a manager can be written from scratch or uses the TCPM framework
- `CONFIG_TYPEC_TCPM` consists of helpers to write a manager
  - `tcpm_register_port` registers a port
  - this is used when the TCPC does not support TCPCI
- `CONFIG_TYPEC_TCPCI` consists of helpers to write a manager for controllers
  that support the standard TCPCI (Type-C Port Controller Interface)
  - `tcpci_register_port` registers a port
  - `tcpci_irq` is to be called from the irq handler
  - it also comes with a tiny i2c driver for `tcpci`
- `CONFIG_TYPEC_TCPCI_MT6370` is a driver for MT6370
  - it registers a platform driver for `mt6370-tcpc`
  - it initializes `tcpci_data` and calls `tcpci_register_port`
    - it provides callbacks for `init`, `set_vconn`, and `set_vbus`
    - it has a irq handler that calls `tcpci_irq`
