printk
======

## printk

- many variants depending on which context it is called
- in most cases, it will lock the logbuf and add a record to it by calling
  `vprintk_store`
- after the record is added, it locks and unlocks the console subsystem
  immediately.  `console_unlock` loops over all new records in logbuf and
  calls `call_console_drivers` to write them to all enabled consoles
  - all mesages are reorded, but `console_loglevel` decides whether to print
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
