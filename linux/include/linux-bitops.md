Bitops
======

# Includes

- `set_bit` is atomic
  - `arch/arm64/include/asm/bitops.h`
    - `asm-generic/bitops/instrumented-atomic.h`
      - `set_bit` calls `arch_set_bit`
    - `include/asm-generic/bitops/atomic.h`
      - `arch_set_bit` calls `raw_atomic_long_or`
    - `include/linux/atomic/atomic-long.h`
      - `raw_atomic_long_fetch_or` calls `raw_atomic64_fetch_or`
    - `linux/atomic/atomic-arch-fallback.h`
      - `raw_atomic64_fetch_or` calls `arch_atomic64_fetch_or`
    - `arch/arm64/include/asm/atomic.h`
      - `ATOMIC64_FETCH_OP` expands to asm instrs
  - `arch/x86/include/asm/bitops.h`
    - `asm-generic/bitops/instrumented-atomic.h`
      - `set_bit` calls `arch_set_bit`
    - `arch/x86/include/asm/bitops.h`
      - `arch_set_bit` generates asm instrs
- `test_bit` is atomic
  - it is defined in non-atomic header because it is a read-only op and does
    not require atomic RMW
  - `linux/bitops.h`: `test_bit` expands to `_test_bit`
  - `arch/arm64/include/asm/bitops.h` includes `asm-generic/bitops/non-atomic.h`
    - `asm-generic/bitops/non-instrumented-non-atomic.h` expands `_test_bit` to `arch_test_bit`
    - `asm-generic/bitops/non-atomic.h` expands `arch_test_bit` to `generic_test_bit`
    - `asm-generic/bitops/generic-non-atomic.h` defines `generic_test_bit`
  - `arch/x86/include/asm/bitops.h` includes `asm-generic/bitops/instrumented-non-atomic.h`
    - `_test_bit` calls `arch_test_bit`
    - `arch_test_bit` calls `variable_test_bit`
    - `variable_test_bit` generates asm instrs

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
