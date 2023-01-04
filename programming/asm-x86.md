x86 Assembly
============

## Stack

- the stack is a piece of memory allocated by the kernel before the program
  runs
  - it is effectively a `MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN` anon map
    whose size is `RLIMIT_STACK` (often 8MB) and who ends near the top of the
    userspace address space (often `1ull<<47` rounded down to page)
  - the kernel uses the stack to pass metadata to the userspace
    - such as those accessible through `argc`, `argv`, `envp`, `getauxval`,
      etc.
  - `rsp` points to the "end" of kernel-generated metadata
    - the stack grows downward so `rsp` is the lowest address of
      kernel-generated data
    - the content at the address is `argc` fwiw
- stack frames
  - a stack frame is a region allocated from the stack
    - it holds the state of a subroutine
    - `rbp + 8` points to the return address when the subroutine returns
    - `rbp` points to the "start" of the previous frame's local variables
    - `[rbp - N, rbp)` point to the current frame's local variables
      - `N` is a fixed size known at compile time
  - a stack frame is constructed by both the caller and the callee
    - the caller sets `rsp` to `rbp - N`, the "end" of the caller's local
      variables
    - when the caller `call`s, the instruction pushes `rip` onto the stack
      - this is the return address
    - the callee pushes `rbp` onto the stack
      - this is the "start" of the caller's local variables
    - the callee sets `rbp` to `rsp`
      - `[rbp - M, rbp]` are used for the callee's local variables
- relevant registers/instructions
  - `rsp` points to "end" of valid data in the stack
    - `push val` is `*(--rsp) = val` in C
    - `pop val` is `val = *(rsp++)` in C
  - `rip` points to the next instruction
    - `call val` is `push %rip; jmp val`
    - `ret` is `pop %rip`
  - `enter` is `push %rbp; mov %rsp, %rbp; sub $N, %rsp`
    - it is never used because it is slower than the 3 instructions combined
  - `leave` is `mov %rbp, %rsp; pop %rbp`
- subroutine parameters
  - the first 6 parameters are passed via registers
    - namely `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`
  - if there are more than 6 parameters, the rest are pushed onto the stack
    before `call`
- subroutine return value
  - it is passed via `rax`
- `-fomit-frame-pointer`
  - use `rsp` for stack frames and free `rbp` up so that `rbp` can be used as
    a general register
- first stack frame
  - gdb backtrace stops at `main` intentionally
  - if looking at the raw stack frame, `main`'s return address is inside
    `__libc_start_call_main`

## Syntax

- `movl 3, %eax` is equivalent to `movl (3), %eax` and is different from
  `mov $3, %eax`.

## EAX

- 32...16...8..0
- EAX is all
- AX is lower 16
- AL is lower 8
- AH is higer 8 of AX

## mov

- On 32-bits,
  - `movl` to `eax` is `B8`, and takes 4 bytes value
  - `movw` to `ax` is `66 B8`, and takes 2 bytes value
  - `movb` to `al` is `B0`, and takes 1 bytes value
- On 16-bits,
  - `movl` to `eax` is `66 B8`, and takes 4 bytes value
  - `movw` to `ax` is `B8`, and takes 2 bytes value
  - `movb` to `al` is `B0`, and takes 1 bytes value

## Jump

- short jump `EB xx`, `xx` is the signed offset from current `ip`
- near jump `E9 xx xx`, `xx xx` is the signed offset from current `ip`
- long jump `EA xx xx yy yy`, `xx xx` is the absolute address and `yy yy` is
  the new segment.  For 32-bits, the absolute address is 4 bytes.
- On 32-bits,
  - `EA` is ljmp and takes 4 bytes address
  - `66 EA` is ljmpw and takes 2 bytes address
- On 16-bits,
  - `EA` is ljmp and takes 2 bytes address
  - `66 EA` is ljmpl and takes 4 bytes address

## `mov` and `lea`

- `movl $123,%eax` is the same as `leal (123),%eax` or `leal 3(120),%eax`
- `leal`, despite its syntax, does not access the memory.  It stores the address
  of the memory, which whould otherwise be accessed by other instructions.

## Kernel syscall

- for kernel syscalls,
  - `%eax` holds the number
  - `%ebx`, `%ecx`, `%edx`, `%esi`, `%edi`, `%ebp` hold the first 6 arguments

## Segmentation

- There are `cs`, `ds`, `ss`, `es`, `fs`, `gs`.
- One may `movl $42, %fs:(%eax)`
- But usually, implied segments are used.
- Instruction fetches use `cs`, code segment.
  - Or, `cs:ip` points to the next instruction
- Most memory references use `ds`, data segment.
- Stack references through `esp`, `ebp`, or `push`, `pop` use `ss`, stack
  segment.
  - Or, `ss:sp` points to the current position of the stack.
- String instructions like `stos` use `es`, extended segment.
  - Or, `ds:si` is copied into `es:di`.
- A segment register is 16-bits
  - 13-bit for index into the descriptor table
  - 2-bit for requested privilege level
  - 1-bit for table indicator
- The privilege check is `DPL < max(CPL, RPL)`
  - `DPL` is the descriptor privilege level of the segment
  - `RPL` is the requested privilege level
  - `CPL` is the current privilege level, which is the `RPL` of `cs`.
  - A `GP` fault is generated if the inequality holds
  - Otherwise, access is granted.
- All, but `cs`, can be modified directly
  - because `cs` gives `CPL`.
- The only way to raise `CPL` is through `lcall` or `int`.
