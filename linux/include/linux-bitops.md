Bitops
======

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
