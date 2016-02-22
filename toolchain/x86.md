x86 Assembly
============

## Syntax

* `movl 3, %eax` is equivalent to `movl (3), %eax` and is different from
  `mov $3, %eax`.

## EAX

* 32...16...8..0
* EAX is all
* AX is lower 16
* AL is lower 8
* AH is higer 8 of AX

## mov

* On 32-bits,
  * `movl` to `eax` is `B8`, and takes 4 bytes value
  * `movw` to `ax` is `66 B8`, and takes 2 bytes value
  * `movb` to `al` is `B0`, and takes 1 bytes value
* On 16-bits,
  * `movl` to `eax` is `66 B8`, and takes 4 bytes value
  * `movw` to `ax` is `B8`, and takes 2 bytes value
  * `movb` to `al` is `B0`, and takes 1 bytes value

## Jump

* short jump `EB xx`, `xx` is the signed offset from current `ip`
* near jump `E9 xx xx`, `xx xx` is the signed offset from current `ip`
* long jump `EA xx xx yy yy`, `xx xx` is the absolute address and `yy yy` is
  the new segment.  For 32-bits, the absolute address is 4 bytes.
* On 32-bits,
  * `EA` is ljmp and takes 4 bytes address
  * `66 EA` is ljmpw and takes 2 bytes address
* On 16-bits,
  * `EA` is ljmp and takes 2 bytes address
  * `66 EA` is ljmpl and takes 4 bytes address

## Instructions

* `push val` is equivalent to `*(--esp) = val` in C
* `call addr` is equivalent to `push eip+2; jmp addr`
* `ret` is equivalent to `pop eip`?
* `enter` is equivalent to `push bp; mov sp,bp`
* `leave` is equivalent to `mov bp,sp; pop bp`

## `mov` and `lea`

* `movl $123,%eax` is the same as `leal (123),%eax` or `leal 3(120),%eax`
* `leal`, despite its syntax, does not access the memory.  It stores the address
  of the memory, which whould otherwise be accessed by other instructions.

## Calling a function

* `push` arguments on the stack
* `call` function
* in function,
  * after entering a function
    * `push ebp`
    * `mov %esp,%ebp`
  * before leaving a function
    * `mov` return value to `%eax`
    * `leave` or `mov %ebp,%esp; pop ebp`
  * `ret`
* 

## Stack

* there is one single big stack per-process.  calling a function usually
  creates a stack frame
* stack frame

      ...
    [ebp + 12] (2nd function argument)
    [ebp + 8]  (1st function argument)
    [ebp + 4]  (return address)
    [ebp]      (old ebp value)
    [ebp - 4]  (1st local variable)
      ...
* `esp` is always made below `ebp` and the function can do anything to the stack
* before a function returns, it clears the stack by
  * `mov %ebp, %esp`
* then retore the `%ebp` by `pop %ebp`
* the above can be replaced by `leave`

## Kernel syscall

* for kernel syscalls,
  * `%eax` holds the number
  * `%ebx`, `%ecx`, `%edx`, `%esi`, `%edi`, `%ebp` hold the first 6 arguments

## Segmentation

* There are `cs`, `ds`, `ss`, `es`, `fs`, `gs`.
* One may `movl $42, %fs:(%eax)`
* But usually, implied segments are used.
* Instruction fetches use `cs`, code segment.
  * Or, `cs:ip` points to the next instruction
* Most memory references use `ds`, data segment.
* Stack references through `esp`, `ebp`, or `push`, `pop` use `ss`, stack
  segment.
  * Or, `ss:sp` points to the current position of the stack.
* String instructions like `stos` use `es`, extended segment.
  * Or, `ds:si` is copied into `es:di`.
* A segment register is 16-bits
  * 13-bit for index into the descriptor table
  * 2-bit for requested privilege level
  * 1-bit for table indicator
* The privilege check is `DPL < max(CPL, RPL)`
  * `DPL` is the descriptor privilege level of the segment
  * `RPL` is the requested privilege level
  * `CPL` is the current privilege level, which is the `RPL` of `cs`.
  * A `GP` fault is generated if the inequality holds
  * Otherwise, access is granted.
* All, but `cs`, can be modified directly
  * because `cs` gives `CPL`.
* The only way to raise `CPL` is through `lcall` or `int`.
