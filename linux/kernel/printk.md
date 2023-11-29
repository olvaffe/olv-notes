printk
======

## printk

- many variants depending on which context it is called
- in most cases, it will lock the logbuf and add a record to it by calling
  `vprintk_store`
- after the record is added, it locks and unlocks the console subsystem
  immediately.  `console_unlock` loops over all new records in logbuf and
  calls `call_console_drivers` to write them to all enabled consoles
  - all mesages are recorded, but `console_loglevel` decides whether to print
    to consoles or not
- it also calls `wake_up_klogd` to schedule a wakeup
  - `log_wait` is wait queue head that allows userspace to poll
  - syslogd incorporates a function called klogd which polls /proc/kmsg
- syslog(2) syscall
  - unrelated to syscall(3)
  - to read logbuf, clear logbuf, change log level, etc.
- /proc/sys/kernel/printk (kernel/sysctl.c)
  - change log level, etc.
- /proc/kmsg (fs/proc/kmsg.c)
  - read-only
  - equivalent to `syslog(SYSLOG_ACTION_READ)`
- /dev/kmsg (drivers/char/mem.c)
  - read or inject records
- subsystems can also register kmsg dumpers which are called on oops
  - this allows saving the logbuf to a persistent storage on crashing

## Console

- sinks for timely printk outputs
- `register_console` is used to register a console
  - most serial port drivers can register a console
    - e.g., `CONFIG_SERIAL_8250_CONSOLE`
  - pstore registers a console when `CONFIG_PSTORE_CONSOLE`
  - kgdb registers a console
  - VT registers a console when `CONFIG_VT_CONSOLE`
- those specified in `console=` cmdline are enabled
  - `console_setup` parses each `console=` and saves them to `console_cmdline`
    array in order
  - each console is parsed into `name`, `index`, and `options`
  - `preferred_console` is set to the last console
- `try_enable_preferred_console` enables consoles
  - if no `console=`, `try_enable_default_console`
  - `printk` prints to all consoles
  - `preferred_console` points to the last console 
    - it will have `CON_CONSDEV` bit, meaning `/dev/console` refers to this
      console

## netconsole

- specify `netconsole=@192.168.0.2/,@192.168.0.1/` in cmdline
- host does `nc -u -l 6666`

## Config

- enable `CONFIG_DYNAMIC_DEBUG` for all `pr_debug` messages
  - dynamically configurable
  - `dyndbg="+p"` for cmdline
  - `/sys/kernel/debug/dynamic_debug/control`
- statically enabling `pr_debug` messages
  - `CONFIG_DEBUG_DRIVER` for driver core
  - `CONFIG_DEBUG_KOBJECT` for kobject
  - `CONFIG_PCI_DEBUG` for PCI
  - `CONFIG_DEBUG_GPIO` for GPIO
  - `CONFIG_I2C_DEBUG_{CORE,ALGO,BUS}` for I2C
  - `CONFIG_HWMON_DEBUG_CHIP` for hardware monitor
  - `CONFIG_DEBUG_PINCTRL` for pin control
  - `CONFIG_POWER_SUPPLY_DEBUG` for power supply
  - `CONFIG_REGULATOR_DEBUG` for regulator
  - `CONFIG_RTC_DEBUG` for RTC
  - `CONFIG_SPI_DEBUG` for SPI
