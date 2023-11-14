CrOS Metrics
============

## Crash Reporter

- <https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/crash-reporter/README.md>
- crash report storage
  - early boot
    - `/run/crash_reporter/crash`
    - `/mnt/stateful_partition/unencrypted/preserve/crash`
  - test build or before login
    - `/var/spool/crash` for system reports
    - `/home/chronos/crash` for user reports
  - user build after login
    - `/run/daemon-store/crash/<user_hash>`
  - `crash_reporter` writes reports to the storage
  - `crash_sender` uploads reports from the storage and removes them
- init scripts
  - `crash-reporter-early-init.conf` runs `crash_reporter --init --early`
    before stateful is mount
  - `crash-reporter.conf` runs `crash_reporter --init` after stateful is mount
  - `crash-boot-collect.conf` runs `crash_reporter --boot_collect` to gather
    crashes from previous boots (e.g., for kernel crashes)
  - `crash-sender-login.conf` runs `crash_sender` once on user login
  - `crash-sender.conf` runs `crash_sender` every 1 hour
  - `anomaly-detector.conf` runs `anomaly_detector`
    - it scans syslog for anomalies that are not crashes
- `crash_reporter`
  - on boot, it is invoked with `--init`
    - it calls `InitializeSystem` and returns
    - `UserCollector::SetUpInternal` writes `/proc/sys/kernel/core_pattern` to
      tell kernel to invoke `crash_reporter` with certain flags on coredumps
  - also on boot, it is invoked with `--boot_collect`
    - it calls `BootCollect` and returns
    - it writes crash reports for crashes from prior boots
  - it is invoked on various events
    - on coredumps, it is invoked by the kernel
      - if chrome crashes, the invocation is actually ignored.  chrome will
        take care of pre-processing before invoking `crash_reporter`
    - on certain udev events (`99-crash-reporter.rules`), it is invoked by
      udev
- crash collectors
  - <https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/crash-reporter/docs/collectors.md>
  - each colloector is a submodule of `crash_reporter`
  - severity
    - `info` if it is not a real crash
    - `warning` if there is little user impact
    - `error` if there is tangible user impact (e.g., tab crash, android app
      crash)
    - `fatal` if there is significant user impact (e.g., session termination,
      device reboot, android system server crash)
  - Boot Collectors
    - these are run once at boot time by `crash-boot-collect.conf`
    - bert, ec, gsc, kernel (ramoops) collectors
  - Runtime Collectors
    - arcpp/arcvm java, arcpp cxx, arcvm cxx/kernel
    - chrome (this has been pre-processed by chrome itself)
    - mount, udev
    - user (coredumps)
    - vm
      - this uses `/home/root/<user_hash>/crash` for storage
  - Anomaly Detectors
- crash report metadata
  - when writing the crash report, each collector writes a `.meta` file for
    metadata
  - Product Names
    - user collector uses `ChromeOS`
    - arc collectors use `ChromeOS_ARC`
    - chrome uses `ChromeCrashReporterClient::GetProductNameAndVersion` to
      determine the product name
      - `Chrome_ChromeOS` if cros
      - `Chrome_Lacros` if lacros
      - `Chrome_Linux` if linux
      - `Chrome_Android` if android
