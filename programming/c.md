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
