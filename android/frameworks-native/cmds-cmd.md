Android `cmd`
=============

## Usage

- `cmd` interacts with `defaultServiceManager()`
- `cmd -l` calls `sm->listServices` to list all services
- `cmd <service> [args...]`
  - `sm->checkService` returns the specified service
  - `IBinder::shellCommand` sends `SHELL_COMMAND_TRANSACTION` to the service

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
