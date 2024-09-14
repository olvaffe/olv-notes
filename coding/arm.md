ARM Assembly
============

## Resources

- guides
  - <https://developer.arm.com/documentation/102374/latest/>
- references
  - <https://developer.arm.com/documentation/ddi0487/latest>
  - <https://developer.arm.com/documentation/ddi0601/latest>
  - <https://developer.arm.com/documentation/ddi0602/latest>

## Registers

- 31 64-bit general-purpose registers
  - `X0..X30` to use as 64-bit registers
  - `W0..W30` to use as 32-bit registers
- 32 128-bit floating-point registers
  - `Q0..Q127` to use as 128-bit registers
  - `D0..D127` to use as 64-bit registers
  - `S0..S127` to use as 32-bit registers
  - `H0..H127` to use as 16-bit registers
  - `B0..B127` to use as 8-bit registers
- 2 zero registers
  - always read as 0 and ignore writes
  - `XZR` to use as 64-bit
  - `WZR` to use as 32-bit
- 1 stack pointer register
  - `SP`
- `X30` can be used as the link register
  - referred as `LR`
- 1 program counter register
  - `PC`
  - not general-purpose
  - can be read using `adr Xn, .`
- system registers
  - `mrs Xn, system-reg` to load
  - `msr system-reg, Xn` to store

## Load and Store

- `ldr`
  - `ldr Wn, address-mode` loads 4 bytes from addr
  - `ldr Xn, address-mode` loads 8 bytes to addr
- `str`
  - `str Wn, address-mode` stores 4 bytes from addr
  - `str Xn, address-mode` stores 8 bytes to addr
- address-mode
  - `[Xn]` where the addr is from `Xn`
  - `[Xn, #offset]`, where `offset` must fall in one of the rules
    - `[-256, 255]`,
    - `[0, 16380]` and multiples of 4 if 32-bit,
    - `[0, 32760]` and multiples of 8 if 64-bit,
  - `[Xn, Xm]`, where `Xm` provides the offset
  - `[Xn, Xm, lsl #s]`, where `s` left-shits `Xm`
- `ldp W3, W7, [X0]` loads a pair
  - `ldr W3, [X0]` and `ldr W7, [X0, #4]`
- `stp D0, D1, [X4]` stores a pair
  - `str D0, [X0]` and `str D1, [X0, #8]`

## General Instructions

- `orr (immediate)`
  - `orr Xn, Xm, #imm`
- `orr (shifted register)`
  - `orr Xn, Xm, Xl{, shift #s}`
  - `Xl` can be shifted by `s`
- `lsl (immediate)`
  - `lsl Xn, Xm, #s`
- `lsl (register)`
  - `lsl Xn, Xm, Xl`
- `bic`: bit clear
  - e.g., `bic r0, r1, #0x1`
