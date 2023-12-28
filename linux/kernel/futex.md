Linux futex
===========

## Futexes

- futex(7)
- a futex is an atomic int in userspace
- when the userspace `FUTEX_WAIT` for the atomic int to change away from a
  specified value
  - kernel sets up a `futex_q` and adds it `futex_queues`
- when the userspace changes the atomic int and `FUTEX_WAKE`
  - kernel find the `futex_q` and wakes up the waiter
- kernel uses the address of the atomic int as the key of `futex_queues`
  - when the atomic int is private, the address can be used directly
  - when the atomic int is shared, shmem inode and offset are used instead
- a futex is thus a userspace atomic int with an associated kernel wait queue
