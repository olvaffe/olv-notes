Kernel Watchdog
===============

## Watchdog Framework

- a watchdog hw resets the system if it times out
  - pinging a watchdog hw resets the timeout
- in the probe function, the driver inits out `watchdog_device` and calls
  `watchdog_register_device` to register the device
  - `watchdog_dev_register` does the registration
    - this registers `/dev/watchdog%d`
    - this also registers `/dev/watchdog` which is the same as
      `/dev/watchdog0`
