# RCU

- RCU stands for read, copy, and update
- A sync mechanism that is optimized for read-mostly situations
- reader
  - `rcu_read_lock`
  - `rcu_dereference`
  - `rcu_read_unlock`
  - very lightweight lock or no locking at all
- writer
  - `rcu_assign_pointer` to update a pointer
  - `synchronize_rcu` to reclaim/free the objects pointed by the old pointers
