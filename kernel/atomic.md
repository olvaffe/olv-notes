# Atomic

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

## x86 bitops.h

- `set_bit` is `lock; orb imm,dst` or `lock; btsq src,dst`
- `clear_bit` is `lock; andb imm,dst` or `lock; btrq src,dst`
- `clear_bit_unlock` is `barrier` followed by `clear_bit`
- `clear_bit_unlock_is_negative_byte` is `lock; andb imm,dst; sets dst2`
- `change_bit` is `lock; xorb imm,dst` or `lock; btcq src,dst`
- `test_and_set_bit` is `lock; btsq src,dst; jc label`
- `test_and_set_bit_lock` is `test_and_set_bit`
- `test_and_clear_bit` is `lock; btrq src,dst; jc label`
- `test_and_change_bit` is `lock; btcq src,dst; jc label`
- `test_bit` is `lock; btq src,dst; jc label` or logical AND
