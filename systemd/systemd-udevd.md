udev
====

## `udevd`

- `manager_main`
  - `manager_setup_event` listens to signals, psi, watchdog, etc.
  - `manager_start_ctrl` listens to a ctrl socket to take cmds from `udevadm`
    - `on_ctrl_msg` handles the cmds
  - `manager_start_varlink_server` listens to a varlink socket to take cmds
    from `udevadm`
    - `sd_varlink_server_bind_method_many` binds the handlers
    - varlink is the new interface that deprecates ctrl
  - `manager_start_device_monitor` listens to the kernel uevent socket
    - `device_monitor_new_full` creates `AF_NETLINK`/`NETLINK_KOBJECT_UEVENT`
      socket
    - `on_uevent` handles uevents
  - `manager_start_inotify` creates an inotify fd to watch devices
    - this seems to be for block devices to re-read partition tables
      - see `/run/udev/watch` for watched devices
    - `manager_add_watch` watches `IN_CLOSE_WRITE` on `sd_device_get_devname`
      (`/dev/foo`)
    - `on_inotify` handles file changes
  - `manager_start_worker_notify` listens to a socket for worker notifies
    - `udevd` forks workers which can notify `udevd` via `sd_notify`
    - `on_worker_notify` handles worker notifies
- when there is a kernel uevent,
  - `device_monitor_event_handler` creates a temp `sd_device` for the uevent
    and calls `on_uevent`
    - `event_queue_insert` wraps the device in an `Event` and adds the event
      to `manager->events`
  - `on_post` calls `event_queue_start` to spawn workers and process events
    - if there is no idle worker, `worker_spawn` forks a new worker to execute
      `udev_worker_main` and process the event device
    - if there is an idle worker, `device_monitor_send` sends the event device
      to the worker
- `udev_worker_main` handles kernel uevents
  - `sd_device_monitor_start` monitors for new event devices to process
  - `worker_device_monitor_handler` handles the initial and subsequent event devices
    - `udev_event_execute_rules` executes the rules
    - `udev_event_execute_run` executes `RUN=`
    - `device_update_db` updates db
    - `device_monitor_send` notifies libudev listeners (e.g., `udevadm`)
    - `sd_notify` sends worker notify to `udevd`
- `udev_event_execute_rules` executes the rules
  - `udev_rules_apply_to_event` calls `udev_rule_apply_line_to_event` to apply
    a line, which calls `udev_rule_apply_token_to_event` to apply a token
  - `TK_M_*` are match tokens
    - `TK_M_ACTION` matches `sd_device_get_action`
    - `TK_M_DEVPATH` matches `sd_device_get_devpath`
    - `TK_M_KERNEL` matches `sd_device_get_sysname`
    - `TK_M_DEVLINK` matches `FOREACH_DEVICE_DEVLINK`
    - `TK_M_NAME` matches `event->name`
    - `TK_M_ENV` matches `device_get_property_value_with_fallback`
    - `TK_M_CONST` matches global consts
    - `TK_M_TAG` matches `FOREACH_DEVICE_CURRENT_TAG`
    - `TK_M_SUBSYSTEM` matches `sd_device_get_subsystem`
    - `TK_M_DRIVER` matches `sd_device_get_driver`
    - `TK_M_ATTR` matches `sd_device_get_sysattr_value`
    - `TK_M_SYSCTL` matches `sysctl_read`
    - `TK_M_TEST` matches `stat`
    - `TK_M_PROGRAM` matches prog execution
    - `TK_M_IMPORT_FILE` imports a file and calls `device_add_property`
    - `TK_M_IMPORT_PROGRAM` imports prog stdout and calls `device_add_property`
    - `TK_M_IMPORT_BUILTIN` runs a builtin cmd
    - `TK_M_IMPORT_DB` imports db and calls `device_add_property`
    - `TK_M_IMPORT_CMDLINE` imports cmdline and calls `device_add_property`
    - `TK_M_IMPORT_PARENT` imports parent and calls `device_add_property`
    - `TK_M_RESULT` matches `event->program_result` from `TK_M_PROGRAM`
  - `TK_A_*` are assign tokens
    - `TK_A_OPTIONS_*` sets event options
    - `TK_A_OWNER` updates `event->uid`
    - `TK_A_GROUP` updates `event->gid`
    - `TK_A_MODE` updates `event->mode`
    - `TK_A_SECLABEL` updates `event->seclabel_list`
    - `TK_A_ENV` calls `device_add_property`
    - `TK_A_TAG` calls `device_add_tag`
    - `TK_A_NAME` updates `event->name`
    - `TK_A_DEVLINK` calls `device_add_devlink`/`device_remove_devlink`
    - `TK_A_ATTR` calls `sd_device_set_sysattr_value`
    - `TK_A_SYSCTL` calls `sysctl_write`
    - `TK_A_RUN_BUILTIN` updates `event->run_list`
    - `TK_A_RUN_PROGRAM` updates `event->run_list`

## Built-ins

- `udev_builtin_run` executes a builtin
- `udev_builtin_blkid` probes a block device with libblkid and calls
  `udev_builtin_add_property` for each blkid key/val pairs
  - `ID_FS_*`, such as `TYPE`, `UUID`, `LABEL`, `FSSIZE`, etc.
  - `ID_PART_TABLE_*`, such as `TYPE`, `UUID`, etc.
  - `ID_PART_ENTRY_*`, such as `NAME`, `TYPE`, etc.
  - if loopback, `ID_LOOP_BACKING_*`, such as `DEVICE`, `INODE`, `FILENAME`, etc.
  - if gpt, `ID_PART_GPT_*`
- `udev_builtin_btrfs` emits `BTRFS_IOC_DEVICES_READY` ioctl and adds
  `ID_BTRFS_READY` prop for block devices with btrfs
- `udev_builtin_dissect_image` adds `ID_DISSECT_*` props for loopback block
  devices
- `udev_builtin_factory_reset` adds `ID_FACTORY_RESET` prop
- `udev_builtin_hwdb` adds props from hwdb
- `udev_builtin_input_id` parses sysfs attrs and adds `ID_INPUT_*` props
- `udev_builtin_keyboard` parses props from hwdb and inits input devices
  - it maps `KEYBOARD_KEY_*` to `EVIOCSKEYCODE`
  - it maps `EVDEV_ABS_*` to `EVIOCSABS`
- `udev_builtin_kmod` loads modules with kmod based on `MODALIAS` prop
- `udev_builtin_net_driver` adds `ID_NET_DRIVER` prop
- `udev_builtin_net_id` adds `ID_NET_NAMING_SCHEME` and `ID_NET_NAME_*` props
- `udev_builtin_net_setup_link` applies net `*.link` (`man systemd.link`)
- `udev_builtin_path_id` adds `ID_PATH*` props
  - they are used to create `/dev/foo/by-path/*`
- `udev_builtin_uaccess` updates acl with libacl for seat devices
- `udev_builtin_usb_id` probes usb devices and adds `ID_BUS`, `ID_MODEL`,
  `ID_SERIAL`, `ID_VENDOR`, `ID_USB_*`, etc.

## Device Properties

- upon kernel uevent, `device_new_from_nulstr` creates a device from the uevent
  - `device_set_devnum` is from `MAJOR` and `MINOR`
  - `device_set_syspath` is from `DEVPATH` (prefixed by `/sys`)
  - `device_set_subsystem` is from `SUBSYSTEM`
  - `device_set_devtype` is from `DEVTYPE`
  - `device_set_devname` is from `DEVNAME` (prefixed by `/dev`)
  - `device_set_usec_initialized` is from `USEC_INITIALIZED`
  - `device_set_driver` is from `DRIVER`
  - `device_set_ifindex` is from `IFINDEX`
  - `device_set_devmode` is from `DEVMODE`
  - `device_set_devuid` is from `DEVUID`
  - `device_set_devgid` is from `DEVGID`
  - `device_set_action_from_string` is from `ACTION`
  - `device_set_seqnum` is from `SEQNUM`
  - `device_set_diskseq` is from `DISKSEQ`
  - `device_add_devlink` is from `DEVLINKS`
  - `device_add_tag` is from `TAGS` and `CURRENT_TAGS`
  - `device->database_version` is from `UDEV_DATABASE_VERSION`
  - note that `device_set_*` also calls `device_add_property_internal` to add
    the uevent key/val as a property
- `udev_event_execute_rules` can call
  - `device_add_property`
  - `device_add_tag`
  - `device_add_devlink`
- `udevadm info /sys/class/net/wlan0`
  - `P: /devices/pci0000:00/0000:00:14.3/net/wlan0` is from `sd_device_get_devpath`
  - `M: wlan0` is from `sd_device_get_sysname`
  - `R: 0` is from `sd_device_get_sysnum`
  - `J: n3` is from `sd_device_get_device_id`
  - `U: net` is from `sd_device_get_subsystem`
  - `T: wlan` is from `sd_device_get_devtype`
  - `I: 3` is from `sd_device_get_ifindex`
  - `E: KEY=VAL` is from `FOREACH_DEVICE_PROPERTY`
    - `device_properties_prepare` calls `device_add_property_internal` for
      devlinks and tags to turn them into props

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
