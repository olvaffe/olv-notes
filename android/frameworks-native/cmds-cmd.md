# Android `cmd`

## Usage

- `cmd` interacts with `defaultServiceManager()`
- `cmd -l` calls `sm->listServices` to list all services
  - java code calls `ServiceManager.addService` or
    `SystemService.publishBinderService` wrapper to register a service
  - native code calls `defaultServiceManager()->addService` to register
- `cmd <service> [args...]`
  - `sm->checkService` returns the specified service
  - `IBinder::shellCommand` sends `SHELL_COMMAND_TRANSACTION` to the service
- wrappers
  - `am` invokes `cmd activity`
    - it is handled by `ActivityManagerService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`
  - `appops` invokes `cmd appops`
    - it is handled by `AppOpsService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/appop/AppOpsService.java`
  - `device_config` invokes `cmd device_config`
    - it is handled by `DeviceConfigService::onShellCommand` of
      `frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java`
  - `dpm` invokes `cmd device_policy`
    - it is handled by `DevicePolicyManagerService::onShellCommand` of
      `frameworks/base/services/devicepolicy/java/com/android/server/devicepolicy/DevicePolicyManagerService.java`
  - `ime` invokes `cmd input_method ime`
    - it is handled by `InputMethodManagerService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/inputmethod/InputMethodManagerService.java`
  - `input` invokes `cmd input`
    - it is handled by `InputManagerService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/input/InputManagerService.java`
  - `locksettings` invokes `cmd lock_settings`
    - it is handled by `LockSettingsService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/locksettings/LockSettingsService.java`
  - `pm` invokes `cmd package`
    - it is handled by `PackageManagerService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`
  - `settings` invokes `cmd settings`
    - it is handled by `SettingsService::onShellCommand` of
      `frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsService.java`
  - `wm` invokes `cmd window`
    - it is handled by `WindowManagerService::onShellCommand` of
      `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java`

## Service

- take native `gpuservice` service as an example
  - it calls `sm->addService` to add itself as a service
  - `cmd` sends `SHELL_COMMAND_TRANSACTION`
  - `BnGpuService::onTransact` calls `GpuService::shellCommand` to handle the
    transaction
- take java `activity` service as an example
  - `ActivityManagerService::setSystemProcess` adds itself as a service
    - it also adds several other services such as `meminfo`, `gfxinfo`, etc.
  - `cmd` sends `SHELL_COMMAND_TRANSACTION`
  - `ActivityManagerService::onTransact` calls
    `ActivityManagerService::onShellCommand` indirectly to handle the transaction
    - there are `Binder::onTransact -> Binder::shellCommand` in-between
