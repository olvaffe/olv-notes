# Linux Notifier

## Blocking Notifier

- `BLOCKING_INIT_NOTIFIER_HEAD` inits a `blocking_notifier_head`
  - `rwsem` is a `rw_semaphore`
  - `notifier_block` is an ordered list of callbacks
    - sorted by priorities, from high to low
- `blocking_notifier_chain_register` locks the semaphore and adds a callback
  to the ordered list
- `blocking_notifier_chain_unregister` locks the semaphore and removes a
  callback from the ordered list
- `blocking_notifier_call_chain` locks the semaphore and calls each callback
  from the list in order
  - `val` and `v` are passed to the callbacks verbatim
  - it returns the return value of the last called callback
- callback return values
  - `NOTIFY_DONE` means the callback is nop (don't care about this invocation)
  - `NOTIFY_OK` means the callback completes successfully
  - other values (`NOTIFY_OK + errno`, see `notifier_to_errno`) are errors
    - an error without `NOTIFY_STOP_MASK` will be shadowed by further callbacks
  - if `NOTIFY_STOP_MASK` bit is set, no further callback
    - `NOTIFY_STOP` is `NOTIFY_OK|NOTIFY_STOP_MASK`
    - `NOTIFY_BAD` is `notifier_from_errno(-EPERM)`
