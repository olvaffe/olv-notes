# Kernel Locking

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

## Concurrent Programming History

- First Mutual Exclusion Problem and Solution
  - one CPU and one core
  - N processes share a memory and can be scheduled anytime
  - all have critical sections
  - "Solution of a Problem in Concurrent Programming Control", Dijkstra (1965)
    - <https://en.wikipedia.org/wiki/Dekker%27s_algorithm>
- Mutexes and Semaphores
  - "Cooperating sequential processes", Dijkstra (1965)
  - A semaphore is a special integer
    - atomic with a hidden wait queue
    - manipulated with up/down methods
  - A binary semaphore can be used as a mutex to protect critical sections
  - A binary semaphore can be used for signaling
    - A process down(&idle) and kicks up a worker
    - the worker finishes the work and up(&idle)
    - this use is undesirable in modern mutexes as it is error-prone
  - producer/consumer with unbounded buffer
    - producer: while (true)
      - { produce(); down(&mutex); push(); up(&mutex); up(&count); }
    - consumer: while (true)
      - { down(&count); down(&mutex); pop(); up(&mutex); consume(); }
  - rewrite to only use binary semaphores
    - producer: while (true)
      - { produce(); down(&mutex); push(); if (count == 1) up(&ready); up(&mutex); }
    - consumer: while (true)
      - { if (wait) down(&ready); down(&mutex); pop(); wait = (count == 0); up(&mutex); consume(); }
    - count is no longer a semaphore, but a state used by push and pop
- modern rewrite of producer/consumer with unbounded buffer
  - for signaling, semaphores have persistent signals and support both
    wait-before-signal and signal-before-wait
  - condition variables only support wait-before-signal
  - producer: while (true)
    - { produce(); down(&mutex); push(); if (count == 1) signal(&cv); up(&mutex); }
  - consumer: while (true)
    - { down(&mutex); if (count == 0) { wait(&cv, &mutex); } pop(); up(&mutex); consume(); }

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

- problem statement
  - A wants to acquire a set of locks in one order
  - B wants to acquire another set of locks in another order
  - when the two sets of locks overlap, how to guarantee forward progress?
- wait-die algorithm
  - assign timestamps to A and B
  - when A tries to acquire a lock already held by B,
    - if A is older than B, A waits until B releases the lock
    - if A is younger than B, A dies (releases all locks) and retries
  - simple implementation
  - more rollbacks for the younger one
- wound-wait algorithm
  - assign timestamps to A and B
  - when A tries to acquire a lock already held by B,
    - if A is older than B, A wounds B (signals B to die) and waits until B
      releases the block
    - if A is younger than B, A waits until B releases the block
  - complex implementation: requires signaling
  - less rollbacks for the younger one
- setup
  - `DEFINE_WD_CLASS` defines a wait-die `ww_class`
  - `DEFINE_WW_CLASS` defines a wound-wait `ww_class`
  - `ww_mutex_init` inits a mutex for a class
- locking
  - `ww_acquire_init` inits a ctx for a class
    - `ctx->stamp` is initialized to `++ww_class->stamp`
  - `ww_mutex_lock` locks a mutex; if the mutex is already locked,
    - if the class is wait-die, it either waits (older) or dies (younger)
    - if the class is wound-wait, it either wounds (older) or waits (younger)
      - it still waits after wounding
      - while waiting, it dies if wounded
    - if dying, it returns `-EDEADLK`
      - the caller must rollback and unlock all other mutexes
  - `ww_mutex_unlock` unlocks a mutex
  - `ww_acquire_fini`

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

## lockdep

- e.g., `inode->i_lock`
  - each `file_system_type` has `i_lock_key` of type `lock_class_key`
  - each `inode` has `i_lock` of type `spinlock_t`
    - each `spinlock_t` has `dep_map` of type `lockdep_map`
  - when `inode_init_always_gfp` inits an inode,
    - `lockdep_set_class(&inode->i_lock, &sb->s_type->i_lock_key)`
      - all locks of all inodes of the same fs type have the same class
  - `spin_lock` calls `lock_acquire`
    - `register_lock_class` is called on demand
      - it adds a `lock_class` to global `classhash_table`
      - `lock_class_key` is the key to the lock class
    - it adds the lock to `curr->held_locks`, which is an array tracking all
      held locks in locking order
    - `validate_chain` validates `curr->held_locks` (when `CONFIG_PROVE_LOCKING=y`)
      - `check_deadlock` checks if a lock of the same class is held
      - `check_prevs_add` validates and updates the lock graph
        - the class tracks the lock graph
        - if this is a new lock path, this is added to the graph
        - else, this is validated against the graph
  - `spin_unlock` calls `lock_release`
    - it removes the lock from `curr->held_locks`
  - `lockdep_is_held` tests if a lock is held (when `CONFIG_LOCKDEP=y`)
    - it looks up in `curr->held_locks`
- `fs_reclaim_acquire` and `fs_reclaim_release`
  - reclaim has some potential deadlock paths
    - `fs lock; alloc -> direct reclaim -> fs writeback -> fs lock again`
    - `shrinker lock; alloc -> direct reclaim -> shrink -> shrinker lock again`
  - `__fs_reclaim_map` is a static lockdep map
    - it uses itself as the class key
    - note how `register_lock_class` never accesses the class key but only
      uses the pointer as the hash key
  - because the locking paths are not traversed until recliam, we typically
    train lockdep about the paths during init
    - see how `validate_chain` validates and updates the lock graph
