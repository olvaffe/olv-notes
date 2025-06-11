Atomic
======

## Memory Model

- there are multiple CPUs and devices that share the memory system.
- within a CPU, it assumes it owns the memory system exclusively.  It will
  optmize its memory operations to maximize performance
  - e.g., it multi-issues instructions that have no data dependency.  Memory
    operations from these instructions can be in any order.
  - we can assume its order only when there is a clear data dependency
    - overlapping accesses are ordered: when it reads and then writes the same
      memory address, the read is always issued before the write
    - dependent accesses are ordered: when it reads one address and uses the
      result to read another address, the two reads are issued in order.
- A compiler might also assume the compiled program owns the memory system
  exclusively.  It may generate instructions that optimize for the memory
  operations
  - `a = ptr[0]; b = ptr[0]` might generate only one memory load
  - `a = ptr[0]; b = ptr[1]` might generate only one merged memory load
  - `a = *ptr1; b = *ptr2; do_something(b)` might load ptr2 before ptr1 to
    hide the latency from `do_something`
- These can be problematic for CPU-CPU interaction and for device I/O.
- Even when the compiler and the CPU does not reorder, the memory/cache might
  still reorder
  - CPU1 might see outdated value at addr1 and updated value at addr2 even
    with this memory operation order
    - CPU0 issues write to addr1
    - CPU0 issues write to addr2
    - CPU1 issues read from addr2
    - CPU1 issues read from addr1
  - this happens when the cacheline in L1$ of CPU1 for addr2 is invalidated
    but the cacheline for addr1 is not
  - this does not happen on x86
- We need to insert barriers to impose ordering over memory operations, at
  the compiler level, the CPU level, and the memory/cache level

## Memory Barriers

- There are three main types of memory barriers
  - write/store memory barriers, wmb
    - all writes/stores before the barrier must be issued before all
      writes/stores after the barrier
    - it has no effect on reads/loads
    - a CPU can `val = compute(); wmb(); ready = true` to make sure that
      - compiler does not reorder the two writes
      - CPU does not reorder the two writes
      - for another CPU, when `ready` is true, `val` has the valid value
  - read/load memory barriers, rmb
    - all reads/loads before the barrier must be issued before all
      reads/loaders after the barrier
    - it has no effect on writes/stores
    - a CPU can `while (!ready); rmb(); result = val` to make sure that
      - compiler does not reorder the two reads
      - CPU does not reorder the two reads
      - cache invalidation does not reorder
  - general memory barriers, mb
    - all memory operations before the barrier must be issued after
      all memory operations after the barrier
- There are also data dependency barriers, `read_barrier_depends`
  - by definition, when any load before the barrier overlaps with any store
    from another CPU, all stores from the other CPU before that overlapping
    store will be perceptible to any load by this CPU after the barrier
  - rmb implies this barrier
  - imagine
    - CPU0: `val = compute(); wmb(); ready = &val`
    - CPU1: `while (!ready); result = *ready`
  - because of the inherent dependency between the two reads
    - compiler can not reorder
    - CPU can not reorder
    - cache invalidation might reoder!
  - rmb works in this case, but `read_barrier_depends` is supposedly cheaper
- Then there are one-way acquire and release operations
  - An acquire operation, lock or `smp_load_acquire`,  guarantess that all
    memory operations after it happen after it
  - A release operation, unlock or `smp_store_release`,  guarantess that all
    memory operations before it happen before it
  - e.g.,
    - `spin_lock(); val = compute(); ready = true; spin_unlock()`
    - without the acquire/release implication, the two writes can be reordered
      outside of the critical region.

## x86 barriers

- SFENCE

    Performs a serializing operation on all store-to-memory instructions that
    were issued prior the SFENCE instruction. This serializing operation
    guarantees that every store instruction that precedes in program order the
    SFENCE instruction is globally visible before any store instruction that
    follows the SFENCE instruction is globally visible. The SFENCE instruction
    is ordered with respect store instructions, other SFENCE instructions, any
    MFENCE instructions, and any serializing instructions (such as the CPUID
    instruction). It is not ordered with respect to load instructions or the
    LFENCE instruction.

- LFENCE

    Performs a serializing operation on all load-from-memory instructions that
    were issued prior the LFENCE instruction. This serializing operation
    guarantees that every load instruction that precedes in program order the
    LFENCE instruction is globally visible before any load instruction that
    follows the LFENCE instruction is globally visible. The LFENCE instruction
    is ordered with respect to load instructions, other LFENCE instructions,
    any MFENCE instructions, and any serializing instructions (such as the
    CPUID instruction). It is not ordered with respect to store instructions
    or the SFENCE instruction.

- MFENCE

    Performs a serializing operation on all load-from-memory and
    store-to-memory instructions that were issued prior the MFENCE
    instruction. This serializing operation guarantees that every load and
    store instruction that precedes in program order the MFENCE instruction is
    globally visible before any load or store instruction that follows the
    MFENCE instruction is globally visible. The MFENCE instruction is ordered
    with respect to all load and store instructions, other MFENCE
    instructions, any SFENCE and LFENCE instructions, and any serializing
    instructions (such as the CPUID instruction).

- No need for data dependency barriers
- acquire/release disables compiler reordering

## Atomicy

- This is a different issue than ordering, but be aware that concurrent
  `*ptr |= 1` and `*ptr |= 2` from two CPUs might only set one of the two bits

## x86 atomic.h

- inline assembly
  - asm(assembler template 
        : output operands
        : input operands
        : list of clobbered registers);
  - asm can have modifiers
    - `volatile` if it has side-effects
    - `goto` if the asm jumps
      - there will be no output operands
      - there will be labels following the clobbered registers
  - each operand can have modifiers
    - `=` means writeonly
    - `+` means read-write
    - `m` means use memory
    - `r` means use registers
    - `i` means use immediates
- Non-RMW
  - `arch_atomic_read` is `READ_ONCE`
  - `arch_atomic_set` is `WRITE_ONCE`
- RMW w/o return
  - `arch_atomic_add` is `lock; addl src,dst`
    - lock means exclusive ownership of the cache line thus RMW becomes atomic
  - `arch_atomic_sub` is `lock; subl src,dst`
  - `arch_atomic_inc` is `lock; incl dst`
  - `arch_atomic_dec` is `lock; decl dst`
  - `arch_atomic_and` is `lock; andl src,dst`
  - `arch_atomic_or` is `lock; orl src,dst`
  - `arch_atomic_xor` is `lock; xorl src,dst`
- RMW w/ return
  - `arch_atomic_sub_and_test` is
    - bool c = false;
      asm volatile goto("lock; subl src,dst; je label" : ...);
      if (0) {
      label: c = true;
      }
      return c;
    - interrupt between `subl` and `je` is fine because the status register will
      be saved and restored
  - `arch_atomic_dec_and_test` is `lock; decl dst; je label`
  - `arch_atomic_inc_and_test` is `lock; incl dst; je label`
  - `arch_atomic_add_negative` is `lock; addl src,dst; js label`
  - `arch_atomic_add_return` is `lock; xadd src,dst` plus the added value
    - `xadd` swaps src/dst, and then add src to dst
  - `arch_atomic_sub_return` is `arch_atomic_add_return` with the added value
    negated
  - `arch_atomic_fetch_add` is `lock; xadd src,dst`
  - `arch_atomic_fetch_sub` is `lock; xadd src,dst` with negated src
  - `arch_atomic_cmpxchg` is `lock; cmpxchgl src,dst`
    - compare EAX with dst
    - if equal, move src to dst; else move dst to EAX
  - `arch_atomic_try_cmpxchg` is `lock; cmpxchgl src,dst; setz dst2` and more
    - basically, automatically update old to mem value on failures
  - `arch_atomic_xchg` is `lock; xchgl src,dst`
  - `arch_atomic_fetch_and` is
    - `int val = arch_atomic_read(v);`
    - `while (!arch_atomic_try_cmpxchg(v, &val, val & i));`
    - `return val;`
  - `arch_atomic_fetch_or` uses `arch_atomic_try_cmpxchg`
  - `arch_atomic_fetch_xor` uses `arch_atomic_try_cmpxchg`
