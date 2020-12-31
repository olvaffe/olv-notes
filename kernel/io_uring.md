`io_uring`
==========

## Introduction

- <https://kernel.dk/io_uring.pdf>
- synchronous io
  - `read()` and `write()`
  - `pread()` and `pwrite()`
  - `readv()` and `writev()`
  - `preadv()` and `pwritev()`
  - `preadv2()` and `pwritev2()`
- asynchronous io
  - `aio`, mainly `io_setup()`, `io_submit()`, and `io_getevents()`
  - limitations
    - require `O_DIRECT` to bypass page cache
    - may become sync in some conditions
    - hard to use
  - evolution
    - tried and failed
- `io_uring` design
  - design goals, in ascending order of importance
    - easy to use, hard to misuse
    - extendable: created for block storage, but keep network and non-block
      storage in mind
    - feature rich: not just for databases
    - efficiency: while block storage io is 512b or 4kb in size, it might
      change (non-block storage0), or there might be empty requests.
      Per-request overhead should be low
    - scalability: the kernel block layer has a scalable infrastructure.  It
      should be exposed to to applications
  - initial design focused on efficiency
    - zero-copy: submissions and completion events are well-defined structures
      in a shared memory
    - coordination: it was a natural extension to put coordination structures
      in the shared memory
    - lock-free: userspace can't share locking with kernel but has to use
      syscalls, which are costly
    - conclusion: single-procuder single-consumer ring buffer, with clever
      memory ordering and barriers
  - two fundamental operations:
    - request submission
    - associated completion event
    - userspace and kernel change roles for the two operations
    - need a pair of rings, named submission queue (SQ) and completion queue
      (CQ)
  - CQ event structure
    - `struct io_uring_cqe`, completion queue event
      - `__u64 user_data` is carried over from the associated request
      - `__s32 res` is the result of the associated request
        - e.g., the return values of `read()` or `write()`
      - `__u32 flags` is unused
  - SQ event structure
    - it needs to describe a lot of information and be extendable
    - `struct io_uring_sqe`, submission queue entry
      - `__u8 opcode` is the opcode
        - e.g., `IORING_OP_READV`
      - `__u8 flags` is the flags common to all opcodes
      - `__u16 ioprio` is the priority
      - `__s32 fd` is the associated file descriptor
      - `__u64 off` is the offset
      - `__u64 addr` describes the buffer
        - e.g., buf in `read()` or iov in `readv`
      - `__u32 len` describes the size of the buffer
        - e.g., count in `read()` or iovcnt in `readv()`
      - a 32-bit union for opcode-specific flags
        - e.g., flags in `preadv2()`
      - `__u64 user_data` is carried over to `io_uring_cqe`
      - `__u64 pad[3]` for future extension and to pad the struct to 64 bytes
  - CQ ring structure
    - kernel (producer) updates `tail` and userspace (consumer) updates `head`
    - userspace pseudo code
      - `unsigned head = cqring->head;`
      - `read_barrier();`
      - `while (head != cqring->tail)`
        - `struct io_uring_cqe *cqe = cqring->cqes[head & cqring->mask];`
        - `consume(cqe);`
        - `head++;`
      - `cqring->head = head`
      - `write_barrier();`
  - SQ ring structure
    - userspace (producer) updates `tail` and kernel (consumer) updates `head`
    - userspace pseudo code
      - `unsigned tail = sqring->tail;`
      - `unsigned index = tail & (*sqring->ring_mask);`
      - `struct io_uring_sqe *sqe = sqring->sqes[index];`
      - `produce(sqe);`
      - `sqring->array[index] = index;`
      - `tail++;`
      - `write_barrier();`
      - `sqring->tail = tail;`
      - `write_barrier();`
    - there is `sqring->array` for a level of indirection
      - this allows userspace apps submit multiple sqes while allowing them to
        embed `struct io_uring_sqe *` in app-defined structs
  - kernel consumes and retires an sqe before the io request has completed
    - a sqe is ready for reuse once parsed by kernel
    - there can be way more pending io requests in kernel than the size of SQ
    - userspace MUST avoid doing that
    - the size of CQ is by default twice the size of SQ to give userspace some
      flexibility
  - CQ and SQ are independent
    - completion events may arrive in any order
- syscalls
  - `int io_uring_setup(unsigned entries, struct io_uring_params *params)`
    - `entries` is the number of sqes and must be power-of-two
    - returns an fd referencing to the new `io_uring` instance
    - `params` is in/out
      - `sq_entries` and `cq_entries` are out, and specify the real sizes of
      	SQ and CQ
      - `flags`, `sq_thread_cpu`, and `sq_thread_idle` are ins
      - `sq_off` and `cq_off` are outs
    - `mmap(fd, IORING_OFF_SQ_RING)` maps the SQ ring, with various fields at
      offsets specified by `params->sq_off`
      - remember SQ has a level of indirection for sqe array?
      - `mmap(fd, IORING_OFF_SQES)` maps the sqe array
    - `mmap(fd, IORING_OFF_CQ_RING)` maps the CQ ring, with various fields at
      offsets specified by `params->cq_off`
  - `int io_uring_enter(unsigned int fd, unsigned int to_submit,
         unsigned int min_complete, unsigned int flags, sigset_t sig)`
     - `to_submit` is the number of sqes available on SQ
     - `min_complete` has different meanings depending on `fd` and `flags`
       - first of all, `min_complete` is ignored when `IORING_ENTER_GETEVENTS`
       	 is not set
       - when `IORING_ENTER_GETEVENTS` is set, and `fd` was not created with
       	 `IORING_SETUP_IOPOLL`, it waits until there are `min_complete` cqes
       - when `IORING_ENTER_GETEVENTS` is set, and `fd` was created with
       	 `IORING_SETUP_IOPOLL`
       	 - `min_complete == 0` tells kernel to poll and adds cqes for completed
       	   requests immediately
	 - `min_complete != 0` tells kernel to wait until there is at least
	   one cqe in CQ
  - sqes are executed in parallel and can complete out-of-order, unless some
    flags are set
    - when `IOSQE_IO_DRAIN` is set, the sqe is not started until all prior
      sqes have completed successfully
      - e.g., `fsync()` following prior `write()`s
    - when `IOSQE_IO_LINK` is set, the sqe is not started until the prior sqe
      has completed successfully
      - e.g., `write()` following prior `read()` to implement copy
      - the chain of sqes can have indefinite length
      - different chains can still execute in parallel
- SQ memory ordering
  - userspace is producer, reads head, and updates tail
    - userspace fills in some sqes and then submit
    - `io_uring_get_sqe`
      - `__head = atomic_load_explicit(sq->khead, memory_order_acquire)`
      - compare head and tail to know if there is still sqe and return the sqe
      - this allows the sqe to be filled in and can be called multiple times
    - `__io_uring_flush_sq`
      - `atomic_store_explicit(sq->ktail, ..., memory_order_release)`
      - this is called before submit
  - kernel is consumer, reads tail, and updates head
    - kernel parses the sqes and schedules ios
    - `io_submit_sqes`
      - `smp_load_acquire(&rings->sq.tail)` in `io_sqring_entries` to work out
      	the number of sqes
      - `smp_store_release(&rings->sq.head, ctx->cached_sq_head)` in
      	`io_commit_sqring` to update head
  - the userspace release is paired with the kernel acquire
    - writing of sqes in userspace happens before userspace release
    - reading of sqes in kernel happens after kernel acquire
  - the userspace acquire is paired with the kernel release
    - writing of sqes in userspace happens after userspace acquire
    - reading of sqes in kernel happens before kernel release
- CQ memory ordering
  - kernel is producer, reads head, and updates tail
    - kernel is requested to wait for `min_complete` cqes
    - `io_cqring_wait`
      - `smp_rmb()` before `READ_ONCE(rings->cq.head)` in `io_cqring_events`
      	to get the current number of cqes
    - `io_req_complete` is called when a sqe has completed
      - `io_get_cqring` returns a cqe to be filled in
        - there is "an implicit barrier through a control-dependency"
      - `io_commit_cqring` updates cq tail and wakes up waiter
        - `smp_store_release(&rings->cq.tail, ctx->cached_cq_tail);`
  - userspace is consumer, reads tail, and updates head
    - `_io_uring_get_cqe`
      - `__io_uring_peek_cqe`
        - `atomic_load_explicit(ring->cq.ktail, memory_order_acquire)`
      - if no cqe, waits in kernel
    - `io_uring_cqe_seen`
      - `atomic_store_explicit(cq->khead, ..., memory_order_release)`
- questions
  - what to do when CQ or SQ is full?
  - doorbell?
