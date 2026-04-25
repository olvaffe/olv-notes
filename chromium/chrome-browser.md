# Chromium chrome/browser

## Versioning

- `chrome/VERSION`
  - `MAJOR` is incremented every 4 weeks
  - `MINOR` is always 0
  - `BUILD` is incremented every 12 hours
  - `PATCH` is not used on main
- branching
  - when `BUILD` is incremented to `X+1`, it concludes the development of
    BUILD `X`
  - `refs/branch-heads/<X>` branch is created from some earlier commit
    - the branch is not fetched by default
  - a commit to add `chrome_branch_deps.json` is added and tagged on the
    branch
  - `PATCH` version is incremented once a day (or on-demand) if there are
    changes

## Memory Metrics

- `RecordMemoryMetricsAfterDelay` schedules `RecordMemoryMetrics`
- `ProcessMemoryMetricsEmitter::FetchAndEmitProcessMemoryMetrics` collects
  metrics
  - `MemoryInstrumentation::RequestGlobalDump`
    - each process, such as gpu process, calls
      `ClientProcessImpl::RequestOSMemoryDump`
      - `OSMetrics::FillOSMemoryDump` dumps `/proc/<pid>/status`
  - `ProcessMemoryMetricsEmitter::ReceivedMemoryDump`

## Feedback Report

- `chrome/browser/ui/webui/ash/config/chrome_web_ui_configs_chromeos.cc`
  - `RegisterAshChromeWebUIConfigs` adds `OSFeedbackUIConfig`, `OSFeedbackUI`,
    and `ChromeOsFeedbackDelegate`
- `chrome/browser/ash/os_feedback/chrome_os_feedback_delegate.cc`
  - `ChromeOsFeedbackDelegate::PreloadSystemLogs` calls
    `ChromeFeedbackPrivateDelegate::FetchSystemInformation`
  - on `ChromeOsFeedbackDelegate::SendReport`, it creates a `FeedbackData` and
    adds various logs to it
- `system_logs::BuildChromeSystemLogsFetcher`
  - `ChromeInternalLogSource`
    - `CHROME VERSION`
    - `ENTERPRISE_ENROLLED`
    - `about_sync_data`
    - `extensions`
    - `skia_graphite_status`
    - `CHROMEOS_ARC_STATUS`, `CHROMEOS_ARC_POLICY`, etc.
    - `account_type`
    - `demo_mode_config`
    - `monitor_info`
    - `FREE_DISK_SPACE`, `TOTAL_DISK_SPACE`
    - more
  - `CrashIdsSource`
  - `MemoryDetailsLogSource`
    - `mem_usage`
  - `PerformanceLogSource`
    - `high_efficiency_mode_active`
  - `BluetoothLogSource`
    - `CHROMEOS_BLUETOOTH_FLOSS`
  - `CommandLineLogSource`
    - `alsa controls`
    - `cras`
    - `audio_diagnostics`
    - `env`
    - `disk_usage`
  - `DBusLogSource`
    - `dbus_summary` and `dbus_details`
  - `DeviceEventLogSource`
    - `network_event_log`
    - `device_event_log`
  - `IwlwifiDumpChecker`
  - `TouchLogSource`
    - `hack-33025-touchpad`, `hack-33025-touchpad_activity`, etc.
  - `InputEventConverterLogSource`
    - `ozone_evdev_input_event_converters`
  - `ConnectedInputDevicesLogSource`
    - `TOUCHPAD_VENDOR`, `TOUCHPAD_PID`
    - `TOUCHSCREEN_VENDOR`, `TOUCHSCREEN_PID`
    - more
  - `DeviceDataManagerInputDevicesLogSource`
    - `ui_device_data_manager_devices`
  - `TrafficCountersLogSource`
    - `traffic-counters`
  - `DebugDaemonLogSource`
    - `routes` and `routes6`
    - `Profile[*] *`
    - many more from `debugd`
  - `NetworkHealthSource`
    - `network-health-snapshot`, `network-diagnostics`
  - `VirtualKeyboardLogSource`
    - `virtual_keyboard`
  - `AppServiceLogSource`
    - `app_service`
  - `KeyboardInfoLogSource`
    - `keyboard_info`
  - `ShillLogSource`
    - `network_devices` and `network_services`
  - `UiHierarchyLogSource`
    - `UI Hierarchy: *`
