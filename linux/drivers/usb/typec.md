Kernel USB Type-C
=================

## A Type-C Port Needs a Driver?

- controllers usually function autonomously and do not need drivers
  - but simpler controllers might need one
- a driver is also needed to
  - get the port status
  - swap the port power role (to supply or to consume power)
  - swap the port data role (to be host or device)

## Type-C Port Controller Manager

- the os must provide a controller manager if the hw controller is simple
  - e.g., the hw controller does not run its own firmware
- `CONFIG_TYPEC_TCPM` consists of helpers to write a manager
  - `tcpm_register_port` registers a port
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
- there are also drivers that are based on TCPM but are not based on TCPCI
- then there are drivers that are written from scratch

## UCSI (USB Type-C Connector System Software Interface)

- UCSI is a standard interface that the os can use to communicate with a port
  if the port supports the interface
  - it's used to get the port status and to swap the port roles
- `CONFIG_TYPEC_UCSI` is a full UCSI implementation
- `CONFIG_UCSI_ACPI` is a driver for UCSI over ACPI
  - it registers a platform driver for `ucsi_acpi`
  - it calls `ucsi_create` and `ucsi_register`
