## C11

- `aligned_alloc`
- `threads.h`
- `stdatomic.h`
- anonymous structures and unions
  - anonymous structures are not available in C++
- `static_assert`

## C99

- `inline`
- flexible array members
  - array with no size (`[]`) as the last member
- `stdbool.h`
- `stdint.h`
- designated initializers
- compound literals
  - 3 or 3.14 are called integer constants; unchangeable.
  - literals are changeable in C99; such as compound literals here
  - literals are unchangeable in C++ spec
- variadic macros
- `restrict` qual

## Integer Promotion

- Every integer type has an integer conversion rank defined as follows:
  - No two signed integer types shall have the same rank, even if they have
    the same representation.
  - The rank of a signed integer type shall be greater than the rank of any
    signed integer type with less precision.
  - The rank of `long long int` shall be greater than the rank of `long int`,
    which shall be greater than the rank of `int`, which shall be greater than
    the rank of `short int`, which shall be greater than the rank of
    `signed char`.
  - The rank of any unsigned integer type shall equal the rank of the
    corresponding signed integer type, if any.
  - The rank of any standard integer type shall be greater than the rank of
    any extended integer type with the same width.
  - The rank of `char` shall equal the rank of `signed char` and
    `unsigned char`.
  - The rank of `_Bool` shall be less than the rank of all other standard
    integer types.
  - The rank of any enumerated type shall equal the rank of the compatible
    integer type
  - The rank of any extended signed integer type relative to another extended
    signed integer type with the same precision is implementation-defined, but
    still subject to the other rules for determining the integer conversion
    rank.
  - For all integer types T1, T2, and T3, if T1 has greater rank than T2 and
    T2 has greater rank than T3, then T1 has greater rank than T3.
- The following may be used in an expression wherever an int or unsigned int
  may be used:
  - An object or expression with an integer type (other than `int` or
    `unsigned int`) whose integer conversion rank is less than or equal to the
    rank of `int` and `unsigned int`.
  - A bit-field of type `_Bool`, `int`, `signed int`, or `unsigned int`.
- If an `int` can represent all values of the original type (as restricted by
  the width, for a bit-field), the value is converted to an `int`; otherwise,
  it is converted to an `unsigned int`. These are called the integer
  promotions. All other types are unchanged by the integer promotions.

## Arithmetic Conversions

- This pattern is called the usual arithmetic conversions:
  - First, if the corresponding real type of either operand is `long double`,
    the other operand is converted, without change of type domain, to a type
    whose corresponding real type is `long double`.
  - Otherwise, if the corresponding real type of either operand is `double`,
    the other operand is converted, without change of type domain, to a type
    whose corresponding real type is `double`.
  - Otherwise, if the corresponding real type of either operand is `float`,
    the other operand is converted, without change of type domain, to a type
    whose corresponding real type is `float`.
  - Otherwise, the integer promotions are performed on both operands. Then the
    following rules are applied to the promoted operands:
    - If both operands have the same type, then no further conversion is
      needed.
    - Otherwise, if both operands have signed integer types or both have
      unsigned integer types, the operand with the type of lesser integer
      conversion rank is converted to the type of the operand with greater rank.
    - Otherwise, if the operand that has unsigned integer type has rank
      greater or equal to the rank of the type of the other operand, then the
      operand with signed integer type is converted to the type of the operand
      with unsigned integer type.
    - Otherwise, if the type of the operand with signed integer type can
      represent all of the values of the type of the operand with unsigned
      integer type, then the operand with unsigned integer type is converted
      to the type of the operand with signed integer type.
    - Otherwise, both operands are converted to the unsigned integer type
      corresponding to the type of the operand with signed integer type.

## Constants in C

Actually, it's not that complicated:

1) base and suffices choose the possible types.
2) order of types is always the same: int -> unsigned -> long -> unsigned
long -> long long -> unsigned long long
3) we always choose the first type the value would fit into
4) L in suffix == "at least long"
5) LL in suffix == "at least long long"
6) U in suffix == "unsigned"
7) without U in suffix, base 10 == "signed"

That's it.  C90 differs from C99 only in one thing - long long (and LL) isn't
there.  The subtle mess Linus has mentioned is C90 gccism: gcc has allowed
unsigned long for decimal constants, as the last resort.  I.e. if you had
a plain decimal constant that wouldn't fit into long but would fit into
unsigned long, gcc generated a warning and treated it as unsigned long.
C90 would reject the damn thing.  _Bad_ extension, since in C99 the same
constant would be a legitimate signed long long.

But yes, "use the suffix when unsure" is a damn good idea, _especially_ since
the sizeof(long) actually varies between the targets we care about.

## C11 Memory Ordering

- barrier in theory
  - an ordered sequence of two stores in C from CPU1 might be perceived in the
    wrong order by CPU2 for several reasons
    - the compiler might reorder the stores
    - CPU1 might reorder the stores
    - the memory coherency system might reorder the stores
      - e.g., the two stores hits two cachelines of CPU1 local cache, but
      	flushed in the wrong order
  - CPU1 must insert a store barrier between the two stores to ensure the
    correct order
  - let's say CPU1 does the right thing and does
    `store 1->VAL; store barrier; store &VAL->PVAL`, and CPU2 does
    `load PVAL->PTMP; load *PTMP->TMP`
    - is TMP guaranteed to be 1?  No
    - the compiler does not reorder the loads because of data dependency
    - CPU2 does not reorder the loads also because of data dependency
    - however, the memory coherency system might reorder the loads
      - e.g., the first load hits the memory and the second load hits CPU2
      	local cache because of the lack of local cache invalidation
  - CPU2 must insert a data dependency barrier between the two loads to ensure
    the correct order
  - more often, the two loads have no data dependency.  CPU2 must insert a
    load barrier between the two loads to ensure the correct order
    - a load barrier is a stronger version of a data dependency barrier and
      can replace a data dependency barrier
- barrier abstraction: two-way
  - store barrier: stores before or after the barrier cannot be reordered
    across the barrier
  - data dependency barrier: loads with data dependency before or after the
    barrier cannot be reordered across the barrier
    - this seems pointless because the compiler and CPU apparently cannot
      reorder because of data dependency
    - but the memory coherency system can still reorder
      - e.g., the memory coherency system can reorder in ways of CPU local
      	cache flush/invalidation
  - load barrier: loads before or after the barrier cannot be reordered across
    the barrier
  - general barrier: memory accesses (both stores and loads) before or after
    the barrier cannot be reordered across the barrier
- barrier abstraction: one-way
  - acquire operation: all memory operations after the acquire operation
    cannot be reordered before the acquire operation
  - release operation: all memory operations before the release operation
    cannot be reordered after the release operation
  - they are the operations implied by mutex lock/unlcok
- C11 `enum memory_order`
  - an atomic operation can have an explicit memory order
  - `memory_order_relaxed`: the atomic operation is atomic but no ordering
    operation implied
  - `memory_order_consume`: the load operation also performs a consume operation
    - used in release-consume ordering
    - no reads or writes in the current thread with data dependency on the
      atomic variable can be reordered before this load
      - seems pointless because reodering in compiler or CPU is not allowed
      	because of data dependency
      - this is about memory coherency system reordering
	- e.g., it may invalidate the cacheline used by a load following the
	  barrier
    - writes with data dependency in other threads that release the
      same atomic variable are visible in the current thread
    - example
      - `atomic_uintptr_t ptr = NULL; int val = 0;`
      - the other thread does
        - `val = 1; atomic_store_explicit(&ptr, &val, memory_order_release)`
        - is there really a data dependency here?  Probably not at HW level,
          but yes at C spec?
      - the current thread does
        - `int *pval = atomic_load_explicit(&ptr, memory_order_consume)`
        - `if (pval == &val) assert(*pval == 1);
  - `memory_order_acquire`: the load operation also performs an acquire
    operation
    - used in release-acquire ordering
    - no reads or writes in the current thread can be reordered before this
      load
      - `mtx_lock` implies an acquire operation
    - all writes in other threads that releases the same atomic variable are
      visibile in the current thread
  - `memory_order_release`: the store operation also performs a release
    operation
    - any read or write in the current thread cannot be reordered after this
      store
      - `mtx_unlock` implies a release operation 
    - all writes in the current thread are visible in other threads that
      acquire the same atomic variable
    - all writes in the current thread that has a data dependency on the
      atomic variable are visible in other threads that consume the same
      atomic variable
  - `memory_order_acq_rel`: the read-modify-write operation also performs both
    an acquire operation and a release operation
  - `memory_order_seq_cst`: the load/store/read-modify also performs an
    acquire/release/both operation, plus a single total order exists in which
    all threads observe all modifications in the same order
