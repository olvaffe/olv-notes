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

## Core

- `typec_port` corresponds to a physical connector
  - its parent, pdc/tcpc driven by ucsi/tcpm driver, calls
    `typec_register_port` to register a port
- tcpm goes through these states on usb dongle insertion
  - `TOGGLING` is inital state to toggle between power src/snk for detection
  - after insertion, tcpc locks the power snk role and raises an irq
    - `tcpm_cc_change` queues `tcpm_pd_event_handler` to call
      `_tcpm_cc_change` to move to `SRC_ATTACH_WAIT`
  - `SRC_ATTACH_WAIT` moves to `SRC_ATTACHED`, assuming vbus is ready
  - `SRC_ATTACHED` calls `tcpm_src_attach`
    - `tcpm_set_polarity` sets polarity (cc1/cc2)
    - `tcpm_set_vbus` turns on 5v vbus
    - `tcpm_set_roles` sets roles
      - power role is `TYPEC_SOURCE`
      - data role is `TYPEC_HOST`
      - usb roles is `USB_ROLE_HOST`
      - `tcpm_mux_set` updates mux
      - `port->tcpc->set_roles` updates tcpc
      - `typec_set_data_role` and `typec_set_pwr_role` update port
  - tcpc raises another irq when vbus is turned on
    - `tcpm_vbus_change` queues `tcpm_pd_event_handler` to call
      `_tcpm_pd_vbus_on` to move to `SRC_STARTUP`
  - `SRC_STARTUP`
    - if port does not support pd, it transitions to `SRC_READY` directly
    - else, `tcpm_ams_start` starts `POWER_NEGOTIATION` and transitions to
      `SRC_SEND_CAPABILITIES` via `AMS_START`
  - `SRC_SEND_CAPABILITIES` calls `tcpm_pd_send_source_caps`
    - if dongle does not support pd, it transitions to `SRC_READY` after
      timeout
  - tcpc raises another irq when pd response is received
    - `tcpm_pd_receive` queues `tcpm_pd_rx_handler` to transition to
      `SRC_NEGOTIATE_CAPABILITIES`, `SRC_TRANSITION_SUPPLY`, and `SRC_READY`
  - `SRC_READY` calls `tcpm_typec_connect`
    - `typec_register_partner` registers a `typec_partner`, representing the
      usb dongle
    - it queues `tcpm_vdm_discovery_work` to send `CMD_DISCOVER_IDENT`
      - high-end typec cable embeds a chip to identity the cable
      - on cable pd response, `tcpm_pd_receive` queues `tcpm_pd_rx_handler` to
        call `tcpm_handle_vdm_request -> tcpm_pd_svdm -> svdm_consume_identity_sop_prime`
        - `typec_register_cable` registers a `typec_cable`, representing the
          cable
        - `typec_register_cable` registers a `typec_plug`, representing the
          cable chip

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
