# refcount

## `refcount_t`

- an `atomic_t` is a typedef'ed struct with one member, `int`
  - use `atomic_*` such as `atomic_inc` to access an `atomic_t`
- a `refcount_t` is a typedef'ed struct with one member, `atomic_t`
  - use `refcount_*` such as `refcount_inc` to access a `refcount_t`
  - it prints warnings when the refcount saturates or underflows
  - it uses relaxed memory ordering

## `struct kref`

- a `kref` is a struct with one member, `refcount_t`
  - use `kref_*` such as `kref_get` to access a `kref`
  - it has a few `kref_put_*` variants for ease of use
- `kref_init` calls `refcount_set`
- `kref_read` calls `refcount_read`
- `kref_get` calls `refcount_inc`
- `kref_get_unless_zero` calls `refcount_inc_not_zero`
  - this can be used to promote a weak pointer to a strong one
- `kref_put` calls `refcount_dec_and_test`
  - if last ref, it calls the release cb and returns true
- `kref_put_mutex` calls `refcount_dec_and_mutex_lock`
  - if last ref, it calls the release cb with the mutex held and returns true
  - the release cb is expected to unlock the mutex
- `kref_put_lock` calls `refcount_dec_and_lock`
  - if last ref, it calls the release cb with the lock held and returns true
  - the release cb is expected to unlock the spinlock
