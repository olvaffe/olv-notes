# Linux futex

## Overview

- `futex_init` creates 4096-ish `futex_hash_bucket` as global wait queues
- a futex is an atomic int in userspace
  - when there is a need to wait, `FUTEX_WAIT` adds the caller to one of the
    global wait queues identified by the futex
  - when there is a need to wake, `FUTEX_WAKE` wakes up waiters on the global
    wait queue identified by the futex
  - a futex is thus a userspace atomic int with an associated kernel wait
    queue

## futex syscall

- `SYSCALL_DEFINE6(futex, ...)` calls differnt functions based on `op`
- `FUTEX_WAKE` calls `futex_wake`
  - `uaddr` is the addr of the userspace 32-bit int
  - `flags` is typically 0
  - `nr_wake` is the number of waiters to wake
  - `bitset` is `FUTEX_BITSET_MATCH_ANY`
  - `get_futex_key` translates `uaddr` to the unique `futex_key` to identify
    the futex
  - `CLASS(hb, hb)(&key)` calls `futex_hash` to find the `futex_hash_bucket`
    - see `DEFINE_CLASS(hb, ...)`
  - it locks the bucket
  - it loops over the bucket to find the waiters
    - waiters are of type `futex_q`
    - it calls waiters' `wake` callback to add them to `wake_q`
      - `wake` typically points to `futex_wake_mark`
        - `__futex_unqueue` removes the waiter from the bucket
    - it breaks after `nr_wake` waiters
  - it unlocks the bucket
  - `wake_up_q` calls `wake_up_process` to wake up waiters
- `FUTEX_WAIT` calls `futex_wait`
  - `uaddr` is the addr of the userspace 32-bit int
  - `flags` is typically 0
  - `val` is the expected val of `*uaddr`
  - `abs_time` is the timeout
  - `bitset` is `FUTEX_BITSET_MATCH_ANY`
  - `futex_setup_timer` preps a timer if a timeout is specified
  - `futex_wait_setup`
    - `get_futex_key` translates `uaddr` to the unique `futex_key` to identify
      the futex
    - `CLASS(hb, hb)(&key)` calls `futex_hash` to find the `futex_hash_bucket`
    - `futex_q_lock` locks the bucket
    - `futex_get_value_locked` deferences `uaddr`
    - if `*uaddr != val`, it returns `-EWOULDBLOCK`
    - `set_current_state(TASK_INTERRUPTIBLE|TASK_FREEZABLE)` preps the task
      for sleep
    - `futex_queue` adds the entry to the bucket and unlocks the bucket
  - `futex_do_wait`
    - `hrtimer_sleeper_start_expires` arms the timeout timer if any
    - `schedule` sleeps
    - after woken up, `__set_current_state(TASK_RUNNING)`
  - `futex_unqueue` removes the entry from the bucket
    - if already removed by wakeup, return 0
    - else, calls `__futex_unqueue` to remove self and returns 1
  - if timeout, return `-ETIMEDOUT`
  - if interrupted by signal, return `-ERESTARTSYS`

## Userspace Semaphore

- a userspace semaphore can be implmented by a futex
  - `typedef int sem;`
  - `void sem_init(int *sem)`
    - `atomic_set(sem, 0);`
  - `void sem_up(int *sem)`
    - `if (atomic_fetch_inc(sem)) return;`
    - `FUTEX_WAKE(sem, 1);`
  - `void sem_down(int *sem)`
    - `while (true)`
      - `const int val = atomic_dec_if_positive(sem);`
      - `if (val) break;`
      - `FUTEX_WAIT(sem, val);`
- defense against spurious wakeup
  - `sem_down` uses a loop to ensure it decrements an non-zero count
- defense against lost wakeup
  - `FUTEX_WAIT` takes an expected value, which is 0 in our case
    - it checks if the count is still the expected value before adding itself
      to the bucket, with the bucket lock held
  - `FUTEX_WAKE` wakes with the bucket lock held as well
    - if `FUTEX_WAKE` goes first, `FUTEX_WAIT` skips the wait because count is
      non-zero
    - if `FUTEX_WAIT` goes first, it either skips the wait or `FUTEX_WAKE`
      wakes it up

## Userspace Mutex

- a userspace mutex without owner can be implemented by a futex
  - it is essentially a binary semaphore, and is
  - it is not considered a mutex academically
- a userspace mutex with owner can also be implemented by a futex
  - it gains an owner field to make sure only the owner can unlock
  - rather than binary, the state can also be unlocked, locked, and contended
    - this way we call `FUTEX_WAKE` when only contended

## Userspace Condition Variable

- a userspace cv can be implmented by a futex
  - `typedef int cv;`
  - `void cv_init(int *cv)`
    - `atomic_set(cv, 0);`
  - `void cv_wait(int *cv, mutex *mut)`
    - `const int val = atomic_load(cv);`
    - `mutex_unlock(mut)`
    - `FUTEX_WAIT(cv, val);`
    - `mutex_lock(mut)`
  - `void cv_signal(int *cv)`
    - `atomic_inc(cv);`
    - `FUTEX_WAKE(cv, 1);`
  - `void cv_broadcast(int *cv)`
    - `atomic_inc(cv);`
    - `FUTEX_WAKE(cv, INT_MAX);`
- defense against spurious wakeup
  - it is the caller's responsibility to re-check the condition to handle
    spurious wakeup
- defense against lost wakeup
  - `FUTEX_WAIT` takes an expected value, which is the seqno before the wait
    - it checks if the seqno is still the expected value before adding itself
      to the bucket, with the bucket lock held
  - `FUTEX_WAKE` wakes with the bucket lock held as well
    - if `FUTEX_WAKE` goes first, `FUTEX_WAIT` skips the wait because seqno
      has incremented
    - if `FUTEX_WAIT` goes first, it either skips the wait or `FUTEX_WAKE`
      wakes it up
  - besides, these two sequences are atomic to each other
    - signal: `mutex_lock -> update_cond -> cv_singal -> mutex_unlock`
    - wait: `mutex_lock -> check_cond -> cv_wait (implies mutex_lock)`
