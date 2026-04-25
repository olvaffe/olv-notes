# ACPI Button

## Power Button

- `acpi_button_probe` probes the acpi device
  - `acpi_install_notify_handler` installs `acpi_button_notify`
- `acpi_button_notify` is called when the button is pressed
  - it reports a `KEY_POWER` input event
  - `acpi_bus_generate_netlink_event` sends a genl event to netlink
  - there used to be `acpi_bus_generate_proc_event` to sends an event to
    `/proc/acpi/event` as well
