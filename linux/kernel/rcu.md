RCU
===

## RCU

- read, copy, and update!
- conceptually simple
  - an RCU-protected pointer, which is a regular pointer, points to an object
  - reader can save the RCU-protected pointer in a local variable lock-free,
    and access the object using the local variable
  - writer can update the RCU-protected pointer lock-free
  - this is nothing special because updating a pointer is atomic; reader gets
    either the old pointer or the new pointer
  - the key is to keep the old object alive until all readers getting the
    older pointer are done with it
- reader
  - use `rcu_read_lock` / `rcu_read_unlock` to mark read-side critical section
    - on a classic implementation, they disable preemption
  - use `rcu_dereference` to access the RCU-protected pointer
    - this simply returns the pointer (with barrier)
- writer
  - `rcu_assign_pointer` to update the RCU-protected pointer
    - this simply updates the pointer (with barrier)
  - `synchronize_rcu` to wait for all readers accessing the old version.  This
    should be called before the writer can free the old version.
    - on a classic implementation, this schedules a dummy task on each CPU.
      Because preemption is disabled in read-side critical section, when the
      dummy task runs, it means the CPU has left the read-side critical
      section.

## SRCU

- srcu stands for sleepable rcu
  - `rcu_read_{lock,unlock}` must obey the same rules as `spin_{lock,unlock}`,
    where blocking and sleeping are prohibited
