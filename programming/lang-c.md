C
=

## C23

- <https://en.cppreference.com/w/c/23>
- single-argument `_Static_assert`
- C++-style function attributes: `nodiscard`, `maybe_unused`, `deprecated`,
  and `fallthrough`
- two's complement sign representation is required
- labels before declarations and at the end of compound statements
- unnamed parameters in function definitions
- binary liberals: 0b10101101 and `%b`
- `#elifdef` and `#elifndef`
- digit separators: 0xFF'FF'FF'FF
- `typeof`
- zero initialization with `{}`
- `alignas`, `alignof`, `bool`, `true`, `false`, `static_assert`,
  `thread_local` become keywords
- no `__VA_OPT__`??

## C17

- a bug-fix release
- `ATOMIC_VAR_INIT` is not needed and obsoleted

## C11

- `stdalign.h`: `aligned_alloc`, `alignof`, etc.
- `stdnoreturn.h`: `noreturn` function specifier to disable some warnings
- `_Generic`
- `threads.h`, optional
- `stdatomic.h`, optional
- anonymous structures and unions
- `assert.h`: `static_assert`
- `time.h`: `timespec`
- VLA becomes optional

## C99

- <https://en.wikipedia.org/wiki/C99>
- `inline`
- mixed declarations and code
- `long long int`, `_Bool`
- variable-length arrays, VLA
- flexible array members
  - array with no size (`[]`) as the last member
- `//`-style comments
- `snprintf`
- `stdbool.h` `stdint.h`, `inttypes.h`
- improved support for IEEE floating point
- designated initializers
- compound literals
  - 3 or 3.14 are called integer constants; unchangeable.
  - literals are changeable in C99; such as compound literals here
  - literals are unchangeable in C++ spec
- variadic macros
- `restrict` qual
- `void foo(int array[static 10])`
- `__func__`
  - there are some that exist before c99
    - `__FILE__`
    - `__LINE__`
    - `__DATE__`
    - `__TIME__`
- `_Pragma(arg)` operator
  - there is already `#pragma` directive before c99
  - c99 also standardizes 3 pragmas
    - `#pragma STDC FENV_ACCESS arg`
    - `#pragma STDC FP_CONTRACT arg`
    - `#pragma STDC CX_LIMITED_RANGE arg`

## Standard Library Headers

- <https://en.cppreference.com/w/c/header>
  - note that POSIX adds stuff to the std headers as well
- commonly used
  - `assert.h`	Conditionally compiled macro that compares its argument to zero
  - `ctype.h`	Functions to determine the type contained in character data
  - `errno.h`	Macros reporting error conditions
  - `inttypes.h`	Format conversion of integer types
  - `limits.h`	Ranges of integer types
  - `locale.h`	Localization utilities
  - `math.h`	Common mathematics functions
  - `signal.h`	Signal handling
  - `stdalign.h`	alignas and alignof convenience macros
  - `stdarg.h`	Variable arguments
  - `stdatomic.h`	Atomic operations
  - `stdbit.h`	Macros to work with the byte and bit representations of types
  - `stdbool.h`	Macros for boolean type
  - `stddef.h`	Common macro definitions
  - `stdint.h`	Fixed-width integer types
  - `stdio.h`	Input/output
  - `stdlib.h`	General utilities: memory management, program utilities, string conversions, random numbers, algorithms
  - `string.h`	String handling
  - `threads.h`	Thread library
  - `time.h`	Time/date utilities
- less used
  - `complex.h` Complex number arithmetic
  - `fenv.h`    Floating-point environment
  - `float.h`	Limits of floating-point types
  - `iso646.h`	Alternative operator spellings
  - `setjmp.h`	Nonlocal jumps
  - `stdckdint.h`	macros for performing checked integer arithmetic
  - `stdnoreturn.h`	noreturn convenience macro
  - `tgmath.h`	Type-generic math (macros wrapping math.h and complex.h)
  - `uchar.h`	UTF-16 and UTF-32 character utilities
  - `wchar.h`	Extended multibyte and wide character utilities
  - `wctype.h`	Functions to determine the type contained in wide character data

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
        - `if (pval == &val) assert(*pval == 1);`
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

## IEEE 754

- mathematically, not representations in memory,
  - a floating-point format is specified by
    - a base, `b`, which is either 2 or 10
    - a precision, `p`, which is the number of digits
    - an exponent range from `emin` to `emax`, where `emin = 1 - emax` 
  - a positive number is `D.DDDD * 10^e`
    - `D.DDDD` is called significand, coefficient, mantissa, fraction, etc.
    - `e` is called exponent
    - each D is between `[0, b-1]`
    - there is a total of `p` D's
      - in the example shown here, `p` is 5
    - e is between `[emin, emax]`
  - a negative number is `-D.DDDD * 10^e`
  - there are also infinity and NaN
  - numbers with leading zeros have multiple representations
    - `0.1234 * 10^0` and `1.2340 * 10^-1` are the same
    - `0.0000 * 10^e` are the same for any e
  - with b = 10, p = 5, emax = 8
    - emin is -7
    - the largest positive number is `9.9999 * 10^8`
    - the smallest positive number is `0.0001 * 10^-7`
    - the smallest negative number is `-9.9999 * 10^8`
    - the largest negative number is `-0.0001 * 10^-7`
  - non-zero numbers between `(-1^emin, 1^emin)` are called subnormal numbers
    - `0.0001 * 10^-7` is the smallest positive number but is subnormal
    - `1.0000 * 10^-7` is the smallest positive number that is normal
    - this is likely because numbers with leading zeros have multiple
      representations.  When we define a canonical representation for them
      (e.g., the first digit must be 1), a subnormal number has no canonical
      representation
- binary32 memory format
  - bits
    - 1 sign bit
    - 8 exponent bits
    - 23 significand bits
  - mathematically,
    - base is 2
    - precision is 23+1
      - the leading digit is always 1 and is not stored
    - emax is 127
      - the exponent bits can store [0, 255], biased
      - the effective range is [-127, 128]
      - but 128 and -127 are reserved for special numbers
      - `[emin, emax]` is thus `[-126, 127]`
  - special numbers
    - when exponent bits are 0x00
      - mantissa 0 means zero
        - sign decides positive/negative zero
      - mantissa non-zero means subnormal numbers
        - the leading digit of mantissa that is not stored is considered 0
        - exponent bits are treated the same as 0x01 (-126)
    - when exponent bits are 0xff
      - mantissa 0 means infinity
        - sign decides positive/negative infinity
      - mantissa non-zero means NaN
        - ieee does not define what different values of sign or mantissa mean
        - on x86 and arm,
          - `__builtin_nanf()` says mantissa is 0x400000
          - `__builtin_nansf()` says matissa is 0x200000
- binary16 memory format
  - bits
    - 1 sign bit
    - 5 exponent bits
    - 10 significand bits
  - mathematically,
    - base is 2
    - precision is 10+1
    - emax is 15
      - [0, 31] biased
      - [-15, 16] effective
      - 16 and -15 are reserved
  - special numbers follow the same rules
    - arm supports "alternative half-precision" format where 0x1f exponent
      bits are treated normally and does not represent infinity/nan
- revisions
  - IEEE 754-1985
  - IEEE 754-2008
    - new round-to-nearest rule
      - the original rule is round-to-nearest, ties to even
      - added optional round-to-nearest, ties away from zero
    - added fma
  - IEEE 754-2019
- `man fenv`
  - implementation-defined behaviors
    - implmentations are allowed to ignore changes to fenv
    - use `#pragma STDC FENV_ACCESS ON` (or `-frounding-math`)
  - exceptions
    - aarch64
      - `fpsr` is Floating-point Status Register
        - bit0: IOC, invalid operation
        - bit1: DZC, divide-by-zero
        - bit2: OFC, overflow
        - bit3: UFC, underflow
        - bit4: IXC, inexact
      - `mrs`/`msr` can load/store the status reg
      - `feclearexcept`: `mrs`, `bic`, and `msr`
      - `feraiseexcept`: `mrs`, `orr`, and `msr`
      - `fetestexcept`: `mrs` and `and`
      - `fegetexceptflag`: alias to `fetestexcept`
      - `fesetexceptflag`: `feclearexcept` followed by `feraiseexcept`
  - rounding mode
    - aarch64
      - `fpcr` is Floating-point Control Register
      - bit22..23:
        - 0b00: RN (round to nearest)
        - 0b01: RP (round towards plus infinity)
        - 0b10: RM (round towards minus infinity)
        - 0b11: RZ (round towards zero)
      - `fegetround`: `mrs` and `and`
      - `fesetround`: `mrs`, `bic`, `orr`, and `msr`
  - floating-point environment
    - aarch64
      - `fegetenv`: `mrs` both `fpsr` and `fpcr`
      - `fesetenv`: `msr` both `fpsr` and `fpcr`
      - `feholdexcept`: `fegetenv` followed by `feclearexcept`
      - `feupdateenv`: `fegetexceptflag`, `fesetenv`, and `feraiseexcept`
  - rounding functions
    - `rint`/`nearbyint` respects `fesetround`
      - when round-to-nearest, halfway cases round to nearest event
        - `1.5` rounds to `2.0`
        - `2.5` rounds to `2.0`
    - `round` ignores `fesetround`
      - always round-to-nearest, and halfway cases round away from zero
        - `1.5` rounds to `2.0`
        - `2.5` rounds to `3.0`
        - `-1.5` rounds to `-2.0`
    - cast respects `fesetround`
      - consider 4 ints
        - `(1<<24)+0b000`
        - `(1<<24)+0b001`
        - `(1<<24)+0b010`
        - `(1<<24)+0b011`
      - casting them to doubles, the bit 23 and 24 of the significand bits
        (the msb is bit 1) are
        - `0b000`
        - `0b001`
        - `0b010`
        - `0b011`
      - casting them to floats require rounding because floats only have 23
        significand bits
        - bit 24 and after are considered to be after the decimal point for
          rounding purposes
        - `0b000` and `0b010` can be represented exactly because bit 24 is 0
        - `0b001` and `0b011` cannot be represented exactly because bit 24 is 1
          - `FE_UPWARD` rounds them up to `0b010` and `0b100`
          - `FE_DOWNWARD` rounds them down to `0b000` and `0b010`
          - `FE_TOWARDZERO` is the same as `FE_DOWNWARD` because the sign bit
            is cleared
          - `FE_TONEAREST` rounds them to `0b000` and `0b100`
            - the full name is `Round to nearest, ties to even`
              - when it's a tie, round to even numbers
            - because bit 24 is 1 and all bits are it are 0, it is a tie
            - it rounds up or down depending on whether bit 23 is 1 or not
      - similarly, casting doubles to floats goes through the same rounding
    - aarch64
      - `round`: `frinta`
      - `nearbyint`: `frinti`
      - `rint`: `frintx`
      - cast: `fcvt`

## Inline

- <http://gcc.gnu.org/onlinedocs/gcc-4.4.0/gcc/Inline.html>
- quote <http://stackoverflow.com/questions/216510/extern-inline>
- GNU89
  - "inline": the function may be inlined (it's just a hint though). An
    out-of-line version is always emitted and externally visible. Hence you can
    only have such an inline defined in one compilation unit, and every other
    one needs to see it as an out-of-line function (or you'll get duplicate
    symbols at link time).
  - "static inline" will not generate a externally visible out-of-line version,
    though it might generate a file static one. The one-definition rule does not
    apply, since there is never an emitted external symbol nor a call to one.
  - "extern inline" will not generate an out-of-line version, but might call one
    (which you therefore must define in some other compilation unit. The
    one-definition rule applies, though; the out-of-line version must have the
    same code as the inline offered here, in case the compiler call's it instead
    it.
- GNU99
  - "inline": like GNU "extern inline"; no externally visible function is
    emitted, but one might be called and so must exist
  - "extern inline": like GNU "inline": externally visible code is emitted, so
    at most one translation unit can use this.
  - "static inline": like GNU "static inline". This is the only
    portable one between gnu89 and c99
- C99 spec <http://www.greenend.org.uk/rjk/2003/03/inline.html>
  - a function that is always mentioned with `inline` and never with `extern`
  - a function that is mentioned without `inline` or with `extern` somewhere
  - a function that is static inline.
- In practice,
  - Use `static inline` with definition in headers or files.  No others.  This
    would be the best.  For files or files include the headers, it might be
    inlined or not.
  - Follow GNU89 and use `extern inline` with definition in headers.  One of the
    source must define the same function without mentioning inline.  Sources
    calling the funtion might inline it, or call to the one we just defined in
    some file.
  - Follow GNU99 and use `inline` with definition in headers.  It has the same
    effect as GNU89's `extern inline`.  So some file must define it.

## Options for Linking

- `Options for Linking` section of `man gcc`
- the compiler automatically links some standard helpers/libraries
- `-fuse-ld` selects the linker
  - `-fuse-ld=gold` or `-fuse-ld=lld`
- `-nostartfiles` to skip startup files such as `crtbegin.o`
- `-nodefaultlibs` to skip compiler runtime such as `libgcc`
- `-nolibc` to skip c/c++ runtime; that is, skip `libc` or `libstdc++`
- `-nostdlib` implies all above
- `-stdlib` selects the c++ runtime
  - `libstdc++` or `libc++`
- `-static-libstdc++` static links the c++ runtime
- `-static` static links all libraries

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
