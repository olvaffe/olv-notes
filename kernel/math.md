Math
====

## Header Inclusion

- given header H
- if there is an uapi part, that will be in `uapi/H`
- if there is an arch-dependent part, that will be in `asm/X` or `uapi/asm/X`
  - in kernel, `asm/X` is included
  - in userspace, `uapi/asm/X` is included
- `asm/X` can include `asm-gemeric/X` or `uapi/asm-generic/X` for the part
  that is common to all archs

## Types

- `linux/types.h`
  - `uapi/linux/types.h`
    - `uapi/asm/types.h`
      - `asm-generic/bitsperlong.h`
        - `__BITS_PER_LONG` is in uapi
          - always 64 on arm64
          - can be 32 or 64 on x86, depending on 32-bit or 64-bit
        - `BITS_PER_LONG` is not in uapi
          - 64 if 64-bit kernel; 32 otherwise
      - `uapi/asm-generic/int-ll64.h`
         - typedefs `__{s,u}{8,16,32,64}` such as `__u32`
      - `asm-generic/int-ll64.h`
         - typedefs `{s,u}{8,16,32,64}` such as `u32`
    - `uapi/linux/posix_types.h`
      - `uapi/asm-generic/posix_types.h`
        - typedefs `__kernel_{long,ulong,pid,uid,gid,size,off,...}_t`
    - typedefs `__{be,le}{16,32,64}` and `__aligned_{u,be,le}64`
  - typedefs many more types such as
    - `dev_t`, `ino_t`, `pid_t`, `size_t`
    - `u_{char,short,int,long}`
    - `u{nchar,short,int,long}`
    - `{u,u_,}int{8,16,32,64}_t`
    - `dma_addr_t`, `gfp_t`, `phys_addr_t`
    - `atomic_t`, `atomic64_t`, `list_head`
    - etc

## Math

- `linux/math.h`
  - `asm-generic/div64.h`
    - `do_div(a, b)` macro
      - cast `a` to 64-bit
      - cast `b` to 32-bit
      - update `a` in-replace
      - return 32-bit remainder
      - this exists partly because u64 divided by u32 can be optimized on
        32-bit kernel
  - `round_up(x, y)` rounds `x` up to a multiple of `y`, which must be
    power-of-2
  - `roundup(x, y)`, where y can be non-power-of-2
  - `round_down(x, y)` and `rounddown`
  - `DIV_ROUND_UP` divides with roundup
  - `mult_frac(x, numer, denom)` times `x` by a fraction
- `linux/math64.h`
  - `div_{u,s}64` divides {u,s}64 by {u,s}32
  - `div64_{u,s}64` divides {u,s}64 by {u,s}64
  - above are boring except they can be optimized on 32-bit kernel
  - `mul_u32_u32` multiples `u32` by `u32`, returns `u64`
  - `mul_u64_u32_shr` multiples `u64` by `u32`, returns `u64` after shifting
    right the 96-bit result
  - `mul_u64_u32_div` multiples `u64` by `u32`, returns `u64` after dividing
    the 96-bit result by another `u32`
