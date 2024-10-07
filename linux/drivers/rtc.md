Kernel RTC
==========

## Driver

- driver
  - calls `devm_rtc_allocate_device` to alloc an `rtc_device`
  - fills in `rtc_class_ops`
    - `read_time` and `set_time` reads/sets the current time
    - `read_alarm` and `set_alarm` reads/sets an alarm
  - calls `devm_rtc_register_device`
    - `rtc_dev_prepare` creates `/dev/rtc%d`
