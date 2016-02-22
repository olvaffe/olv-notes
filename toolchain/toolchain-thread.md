Thread
======

## Bionic

* In `pthread_create`, a memory to store `struct thread` is malloced.  A stack
  based on anonymous mmap memory is allocated.  The top of the stack is used as
  `tls`.  `clone` is called to fork a new thread.  The parent thread is simple.
  The child thread calls `set_thread_area` to set up gdt/ldt(?) so that `tls` is
  accessible from `%gs:0`.  All threads access their `tls` through `%gs:0` and
  thread-local data is returned.
* `tls[0]` is temporarily used as a mutex until just before `pthread_create`
  returns.  It is used so that the child does not run until the parent unlocks.
