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

## log level

- each message has one of 8 loglevels
  - `LOGLEVEL_EMERG` (0) means system is unusable
  - `LOGLEVEL_ALERT` (1) means action must be taken immediately
  - `LOGLEVEL_CRIT` (2) means critical conditions
  - `LOGLEVEL_ERR` (3) means error conditions
  - `LOGLEVEL_WARNING` (4) means warning conditions
  - `LOGLEVEL_NOTICE` (5) means normal but significant condition
  - `LOGLEVEL_INFO` (6) means informational
  - `LOGLEVEL_DEBUG` (7) means debug-level messages
- `console_printk` is an array of 4 loglevels
  - `console_loglevel` is the current console loglevel
    - `suppress_message_printing` suppresses a message if the message loglevel
      is equal to or greater than `console_loglevel`
    - it is initialized to `CONSOLE_LOGLEVEL_DEFAULT` (7), to print all but
      `LOGLEVEL_DEBUG`
  - `default_message_loglevel` is the defaut message loglevel
    - if a message lacks an explicit loglevel, it is assumed to have
      `default_message_loglevel`
    - it is initialized to `MESSAGE_LOGLEVEL_DEFAULT` (4)
  - `minimum_console_loglevel` is barely used
    - it is used by `SYSLOG_ACTION_CONSOLE_OFF` and
      `SYSLOG_ACTION_CONSOLE_LEVEL`
    - it is initialized to `CONSOLE_LOGLEVEL_MIN` (1)
  - `default_console_loglevel` is not used by the kernel
    - it is such that the userspace knows the default loglevel
    - it is initialized to `CONSOLE_LOGLEVEL_DEFAULT` (7)
- `/proc/sys/kernel/printk` represents the `console_printk` array
  - all 4 numbers can be changed freely by the userspace
- cmdline params
  - `quiet` calls `quiet_kernel` to set `console_loglevel` to
    `CONSOLE_LOGLEVEL_QUIET` (4)
  - `debug` calls `debug_kernel` to set `console_loglevel` to
    `CONSOLE_LOGLEVEL_DEBUG` (10)
  - `loglevel` calls `loglevel` to set `console_loglevel` to the specified
    value
- debug messages
  - a message with `LOGLEVEL_DEBUG` is suppressed by default because
    `CONSOLE_LOGLEVEL_DEFAULT` is also `LOGLEVEL_DEBUG`
  - besides, `pr_debug`, `dev_dbg`, etc are no-op unless `DEBUG` or
    `CONFIG_DYNAMIC_DEBUG` is defined
