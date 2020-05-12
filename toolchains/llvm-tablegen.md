LLVM TableGen
=============

## Overview

- <http://llvm.org/docs/TableGenFundamentals.html>
- `//` or `/* ... */` are comments
- Strongly typed
  - `bit`
  - `int`
  - `string`
  - `bits<n>`
  - `list<ty>`
  - `dag`
  - `code`
- Expressions
  - `"string"`
  - `[{code fragment}]`
  - `[A, B, C]<type>` is a list
  - `{a, b, c}` is an initializer for `bits<3>`
  - `value{15-17}` accesses multiples bits of a value

## `Intrinsics.td`

- `Intrinsics.td` defines all intrinsics
  - it includes `Intrinsics<ARCH>.td`
  - it is used with `-gen-intrinsic` to generate `Intrinsics.gen`
  - the main class is `Intrinsic`, and there is one instance of the class for
    each intrinsic

## `Target.td`

- `Target.td` defines target-independent interface that a target should
  implement
- `SubtargetFeature`
  - In ARM, for example, there are
    - v4t
    - v5t
    - thumb mode
    - vfp2
    - vfp3
    - neon
    - thumb2
    - a8
    - ...
  - a subtarget feature may imply other features
    - a8 implies NeonForFP, VMLxForwarding, ...
- `InstrItinClass`
  - Instructions are grouped into classes.
  - E.g., a class for MUL's, a class for ADD's
- `ProcessorItineraries` and `InstrItinData`
  - the schedule data of an instruction class are specified by `InstrItinData`
  - `ProcessorItineraries` consists of lists of function units, bypasses, and
    instruction itinerary data
  - E.g. In ARMv6, instructions in `IIC_iALUi` class takes 2 cycles.
  - E.g. In Coretex-A9, instructions in `IIC_iALUi` class takes 1 cycle.
- `Processor`
  - describes processors using processor itineraries and subtarget features
- `Register`
  - describes registers of the target
  - A register can have "immediate" sub-registers: `EAX` has `AX`; `AX` has
    `AH` and `AL`
  - On ARM,
    - there are r0 to r12, sp, lr, pc
    - there are s0 to s31 for floats
      * or, d0 to d15 for doubles
      * vfp3 defines additionally d16 to d31
      * or, q0 to q15 for quad-words
    - there are also cpsr, apsr, spsr, fpscr, itstate
    - there are also fpsid, fpexc
- `RegisterClass`
  - groups registers into classes
  - there is GPR for general purpose registers
  - there are many more
- `CallingConv`
  - describes calling conventions
- `Instruction`
  - there are pseudo-instructions, such as `PHI`
- `Target`

## FAQ

- what is `LLVMMatchType` for?
  - Consider `template <typename ty> int foo(int a, ty b, ty c);`.  The type
    of the third argument matches that of the second argument.
    `LLVMMatchType<0>` can be used to describe this.
- what is `LLVMExtendedElementVectorType` for?
  - consider this instruction `vaddhn.i32 i16 i32 i32`, which adds two vectors
    with 32-bit integers, and select the highest halves of the results.  The
    type of the argument is `LLVMExtendedElementVectorType<0>` and the type of
    the result is `llvm_anyvector_ty`
- what is `nocapture` for?
  - no idea
