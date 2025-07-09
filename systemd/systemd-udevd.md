udev
====

## udev rules

- every device is associated with four types of entries
  - properties: `ENV{foo}`
  - devlinks: `SYMLINK`
  - tags
  - sysattr: `ATTRS{foo}`
- In rules,
  - properties are referenced by `ENV{foo}`
  - devlinks are referenced by `SYMLINK`
  - tags are referenced by `TAG{foo}`
  - sysattr are referenced by `ATTRS{foo}`

## `udevd`

- An unix socket for controlling the daemon is created through
  `udev_ctrl_new_from_socket`.
- An netlink socket for monitoring kernel events is created through
  `udev_monitor_new_from_netlink`.
  - the socket returns `struct udev_device`.
  - the device is wrapped in `struct event` and queued on `event_list`.
- An inotify watch is created through `udev_watch_init`.
  - `/lib/udev/rules.d`, `/etc/udev/rules.d`, and `/dev/.udev/rules.d` are added
    to watch list.  Rules are reloaded if changed.
  - some devices are set to be watched.  When a watched device is opened for
    write by any process and is closed, udevd is notified and sends "change" to
    the device's `uevent`.  Check out `/dev/.udev/watch` to see the watched
    devices.
- A signal watch is created through `signalfd`.
- A socket pair is created for talking to the workers.
  - it is used by worker to tell daemon the result of
    `udev_event_execute_rules`.
- workers, idle or active, are stored on `worker_list`.
  - worker is created by `worker_new`.
  - worker processes one `struct udev_event` at a time.  It can process multiple
    events in its lifetime.
  - every worker has a `worker_monitor`.  After processing one event and
    updating the db, it sends the modified device to libudev listeners (e.g.
    udevadm monitor).  It sends the error code of rule execution to the daemon
    through the socket pair.  It then reads the next event from the monitor.
    See `worker_new`.
  - a worker works on a `struct udev_device` by wrapping it in a
    `struct udev_event`.  It thens calls `udev_event_execute_rules` and
    `udev_event_execute_run`.  Rule execution involves `udev_device_update_db`.
- `events_start`
  - for each queued event, it calls `event_run` to find a worker to work on this
    event.
- `struct udev_queue_export` and `struct udev_queue`
  - `udevd` updates current event in `/dev/.udev/queue.bin`.
  - `udevadm settle` can make use of it.

## `udevadm info`

- prefixes
  - `P` Device path in /sys/
    - e.g., `/devices/pci0000:00/0000:00:14.3/net/wlp0s20f3`
  - `M` Device name in /sys/
    - basename of `P`
  - `R` Device number in /sys/
    - numeric suffix of `P`
  - `J` Device ID
  - `U` Kernel subsystem
    - e.g., `net`
  - `B` Driver subsystem
  - `T` Kernel device type within subsystem
    - e.g., `wlan`
  - `D` Kernel device node major/minor
  - `I` Network interface index
    - e.g., `3`
  - `N` Kernel device node name
  - `L` Device node symlink priority
  - `S` Device node symlink
  - `Q` Block device sequence number (DISKSEQ)
  - `V` Attached driver
  - `E` Device property

## `udevadm settle`

- A `struct udev_queue` is created.
- it waits until queue is empty
- `/sys/kernel/uevent_seqnum` gives kernel seqnum, the seqnum of latest event.
- `/dev/.udev/uevent_seqnum` gives the last event udevd has seen.
- `/dev/queue` gives the events being processed
- the above two is deprecated in favor of `/dev/.udev/queue.bin`.
- a queue is empty if there is no busy events and all kernel events have been
  received.

## `vol_id`

- used to add `ID_FS_XXX` properties to block device

## Coldplug and Hotplug

- kernel uevents
  - when a device is hotplugged, the kernel sends a uevent for the new device
  - `systemd-udev-trigger.service` invokes `udevadm trigger` on boot
    - `device_enumerator_scan_devices_and_subsystems` scans `/sys`
    - `sd_device_trigger_with_uuid` calls `sd_device_set_sysattr_value` to write
      `change` to sysfs `uevent` of the device, to trigger a kernel uevent
- `systemd-udevd` has a `sd_device_monitor` to monitor uevents
  - a `sd_device_monitor` monitors `AF_NETLINK` / `NETLINK_KOBJECT_UEVENT`
  - on uevent, `device_monitor_event_handler` is called
    - it calls `on_uevent` provided by udevd which adds the event to
      `manager->events`
  - udevd mainloop calls `event_queue_start` to process `manager->events`
    - `worker_spawn` spawns workers to process events
  - `udev_worker_main` is the entrypoint of a worker
    - `worker_process_device` processes a device
      - `udev_event_execute_rules` executes the rules
        - rule `IMPORT{builtin}="hwdb"` calls `builtin_hwdb`
          - `udev_builtin_hwdb_lookup` adds matching props to the device
      - `udev_event_execute_run` invokes `RUN=`
- udevd drivers.rules does
  - if `MODALIAS` is set in the uevent, invoke `kmod load $env{MODALIAS}`

## `udevadm`

- `udevadm info /sys/class/net/wlan0`
- `udevadm test-builtin net_id  /sys/class/net/wlan0`
  - `man systemd.net-naming-scheme`
  - `man systemd.link`
