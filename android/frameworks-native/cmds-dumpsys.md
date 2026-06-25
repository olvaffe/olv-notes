# Android `dumpsys`

## Internals

- both `dumpsys -l` and `cmd -l` work the same way
  - `sm_->listServices`
- `dumpsys` dumps all services
  - `writeDumpHeader` prints
    - `---------...`
    - `DUMP OF SERVICE <name>:`
  - `startDumpThread` dumps to `redirectFd_`
    - `service->dump` is `BpBinder::dump`
      - it sends `DUMP_TRANSACTION`
      - for comparison, `cmd` sends `SHELL_COMMAND_TRANSACTION`
  - `writeDump` prints `redirectFd_`
  - `writeDumpFooter` prints
    - `--------- <dur> was the duration of dumpsys <name>, ending at: <time>`

## Services

- for native services,
  - `BBinder::transact` calls `BBinder::onTransact` to handle
    `DUMP_TRANSACTION` and call `dump`
  - e.g., `SurfaceFlinger::dump` calls `PriorityDumper::priorityDump` which
    calls `SurfaceFlinger::dumpCritical`
- for java services,
  - `BBinder::transact` calls `JavaBBinderExt::onTransact`
    - `env->CallBooleanMethod` calls java `Binder::execTransact`
    - java `Binder::onTransact` handles `DUMP_TRANSACTION` and calls `dump`
  - e.g., `MemBinder::dump` calls
    `ActivityManagerService::dumpApplicationMemoryUsage`
