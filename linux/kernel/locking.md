Locking
=======

## Synchronizations

- not about the kernel anymore
- see locking.md
- synchronize access to a shared resource
- semaphore
  - up adds a token (of resource readiness)
  - down waits and removes a token (of resource readiness)
  - naive implementation
    - a semaphore is a futex where the int is the number of tokens (usually
      either 0 or 1)
    - up increments the futex, and `FUTEX_WAKE` if was 0
    - down atomically does `FUTEX_WAIT(0)` and decrements the futex
      - in a loop where only one waiter can decrement and exit
- mutex (essentially binary semaphore)
  - lock to acquire ownership (of a resource)
  - unlock to relese ownership (of a resource)
  - naive implementation
    - a mutex is a futex where 0 means unlocked and 1 means locked
    - lock sets the futex to 1 or `FUTEX_WAIT(1)`
      - in a loop where only locker can set and exit
    - unlock sets the futex to 0 and `FUTEX_WAKE`
    - a naive mutex can also be considered a naive binary semaphore
      - initial value is 1
      - lock is down
      - unlock is up
- condition variable
  - wait adds the caller to the wait queue
  - signal wakes up and removes the first caller in the wait queue
  - naive implementation
    - a cv is a list (and a mutex to protect the list)
    - wait adds a futex of value 0 to the list, `FUTEX_WAIT(0)`, and remove
      the futex after signaled
    - signal set the first futex to 1 and `FUTEX_WAKE`
- scenarios
  - N processes with N critical sections
    - use a mutex to protect the critical sections
  - producer/consumer with a bounded buffer
    - need a mutex to protect `produce` and `consume`
    - use two counting semaphores to signal each other
    - or, use two condition variables to signal each other

## Semaphore History

- Problem 1: N processes want to run concurrently but with critical sections
- Solution 1: use a spinlock to protect the critical sections
  - lock contention means wasted CPU cycles
- Idea:
  - lock contention should put processes to sleep
  - unlock should wake them up
- Solution 1a: binary semaphore aka mutex
- Problem 2: producer/consumer with a bounded buffer
- Solution 2: counting semaphore
  - requires 2 counting semaphores and 1 mutex
- For comparison, today
  - we still use mutexes
  - counting semaphores are often replaced by condition variables and ints
  - that is, counters are separated out from counting semaphores, making
    semaphores a pure sync mechanism; mutexes are used to protect the counters

## Locking

- `local_irq_save` to avoid race with current cpu
- `spin_lock` to avoid race with another cpu
- Documentation/spinlocks.txt
- Concurrency: Two or more functions could be called concurrently.
- Reentrancy: The same function could be concurrently called.
- Mutex (kernel/mutex.c)
  - sleep with `__set_task_state(task, TASK_UNINTERRUPTIBLE); schedule();`
  - needs to be waked up

## `spinlock_t`

- `spin_lock_lock` busy waits until the lock is available and then grab it
- `spin_lock_unlock` releases the lock
- On UP, it simply calls `preempt_disable`
- On SMP, the implementation is `__raw_spin_lock`
  - disables preemption
  - calls `queued_spin_lock` to cmpxchg 0 to `_Q_LOCKED_VAL`
  - otherwise, it is contended and it has waiting queues to guarantee fairness
  - in the old days, there was a "ticket spinlock"
    - lock gets a ticket from the higher 8 bits:
      `ticket = post_inc_higher_8_bits(lock)`
    - it waits until the lower 8 bits becomes ticket
    - unlock increments the lower 8 bits
    - that is, the higher bits are "now serving..." number and the lower bits
      are the next number in the ticket dispenser in a store

## `rwlock_t`

- readers-writer spinlock
- On UP, same as `spinlock_t`
- On SMP, writer
  - `write_lock`
    - disables preemption
    - calls `queued_write_lock` to cmpxchg 0 to `_QW_LOCKED`
    - otherwise, there is at least a reader or writer.  A busy loop is
      required.
  - `write_unlock`
- reader
  - `read_lock`
    - disables preemption
    - calls `queued_read_lock` to increment reader count and make sure there
      is no writer
  - `read_unlock`

## `struct mutex`

- defined in `include/linux/mutex.h` and implemented in
  `kernel/locking/mutex.c`
- the `owner` field is a pointer to the `task_struct`, which is the owner of
  the mutex
- `mutex_lock` to acquire the mutex
  - locking an uncontended mutex is a `cmpxchg`
    - set `current` to `owner`
  - locking a mutex locked by another task on another CPU spins a bit
    - the expectation is that the other CPU might unlock the mutex soon so we
      wait a bit
  - otherwise, the current task is added to the waitlist and sleeps
- `mutex_unlock` to release the mutex
  - unlocking an uncontended mutex is also a `cmpxchg`
  - otherwise, wake up waiters

## `struct rw_semaphore`

- readers-writer mutex
- reader
  - `down_read`
  - `up_read`
- writer
  - `down_write`
  - `up_write`

## `struct ww_mutex`

- In database, an atomic transaction locks and updates its data items
  one-by-one
  - when another younger atomic transaction shares some of the data items, and
    attemps to locks and updates those shared data items, there needs to be a
    scheme to avoid deadlock
  - wait-die scheme
    - when the original transaction needs to lock a data item already locked
      by the younger transaction, it waits
    - when the younger transaction needs to lock a data item already locked
      by the original transaction, it rolls back entirely
  - the original transaction starts first and might have locked most of the
    data items by the time the younger transaction starts
    - it is more likely that the younger transaction contends with the
      original transaction than the other way around
    - as such, rollback is more likely
  - wound-wait scheme
    - when the original transaction needs to lock a data item already locked
      by the younger transaction, it requests the younger transaction to roll
      back entirely
    - when the younger transaction needs to lock a data item already locked
      by the original transaction, it waits
  - rollback happens less with wound-wait scheme
    - but waking up younger transactions is heavy
- a `ww_mutex` is initialized with the scheme specified
- `ww_mutex_lock`
  - it takes a `ww_acquire_ctx` as well which is the transaction
  - it either waits (sleeps), dies (returns -EDEADLK), or dies after waiting
    (wounded)
  - when it dies, the caller must release all mutexes for the context to roll
    back
- `ww_mutex_unlock` to release the mutex

## `seqlock_t` and `seqcount_t`

- seqlock is seqcount protected by a spinlock
- seqcount is a sequence number
  - initially 0
  - `write_seqcount_begin` and `write_seqcount_end` both increments the
    sequence number
    - when odd, there is a writer
    - when there can be more than one writers, the seqcount must be protected
      by an external lock
  - `read_seqcount_begin` is a busy loop waiting until there is no writer and
    returns the current sequence number
  - `read_seqcount_retry` checks if there was any writer during reading, and
    if so, the caller needs to read again
- writer
  - `write_seqlock` calls `spin_lock` and `write_seqcount_begin`
  - `write_sequnlock` calls `write_seqcount_end` and `spin_unlock`
- reader
  - `read_seqbegin` calls `read_seqcount_begin`
  - `read_seqretry` calls `read_seqcount_retry`
- this is to prevent DoS
  - an untrusted reader cannot starve the writer by keeping acquiring the
    spinlock
