Position Independent Code
=========================

## PIC

- <http://www.greyhat.ch/lab/downloads/pic.html>
- For functions, relative jumps are used
- For static/global variables or strings, GOT entries are used
  - The GOT entries are derived from `__i686.get_pc_thunk.bx` and an
    offset.  In C, it is `GOT[offset]`, which is the absolute address of the
    variable in current process.  The offset is generated by linker through
    `R_386_GOTPC` to find GOT and `R_386_GOTOFF` to find entry.
  - The same trick can be used locate the variable directly.  Why go through
    GOT?
    * Consider our PIC code is linked to a shared library and the variables it
      access are in that library.  Since the addresses of the variables are
      unknown at link time, we cannot derive the needed offsets to the
      variables.  Therefore, we need GOT.  GOT can be modified by dynamic
      loader.
    * Probably because we do not want to (and cannot in the case of linked to a
      shared library) force text and data sections to have constant offset.
      With GOT, it is GOT that has constant offset from text section.  Data
      section may be put in a better place to share with other processes.
- Now go back to functions again.  What if functions are in some libraries?
  What if we want functions to be overloaded?
  - We cannot, in the former case, and may not want to, in the latter case, call
    the function directly using relative addresses.
  - The soluction is PLT.  In both cases, instead of calling `func`, we call
    `func@plt`.  The first time `func@plt` is called, it calls into dynamic
    loader to locate and store the address of `func` in GOT (`.got.plt`).  Later
    on, `func@plt` can jump to `func` directly.  This also has the benefit of
    lazy binding.
  - By default, global functions of our code are overloadable.  That is, there
    are PLT entries for global functions and they are used.  Static functions
    are not overloadable, no PLT entries for them.  To make global functions not
    overloadable, specify `-Bsymbolic` in ld.
- An executable is usually not PIC.  It can access its functions/variables using
  addresses decided at compile time.
  - How does it access library's variables?  It emits `R_386_COPY` so that
    library's variables are copied to its `.bss`.  It makes them variables _in_
    the executable.
  - How about library's functions?  Just like PLT, with GOT is a known place.
- To call a PIC function `foo`,
  - the caller calls `foo@plt` in section `.plt`
  - `foo@plt` jumps to suitable entry in `.got.plt` directly
  - The entry in `.got.plt` jumps back to `foo@plt+6`, which pushes entry number
    and jumps to the zero-th entry of `.plt`.
  - The zero-th entry of `.plt` is special and it calls dynamic linker to
    resolve the symbol and stores it in `.got.plt`.
  - The next time `foo@plt` is called, it jumps twice to `foo`.
- Here is a sample code

    #include <stdio.h>
    
    static void a(void) { }
    void b(void) { }
    
    int main(void)
    {
    	printf("hello world\n");
    	a();
    	b();
    	return 0;
    }
  - Compile it normally or with `-fPIC`, `-Wl,-Bsymbolic`, etc.  See its
   assembly output!

## References

- <http://www.airs.com/blog/archives/41>

## Old


GOT for static or global variables access

1. Get the offset of GOT

    1d9c:       e8 b6 fb ff ff          call   1957 <__cxa_finalize@plt+0xb3>
    1da1:       81 c3 67 28 01 00       add    $0x12867,%ebx   # the magic number is generated by asm("add _GLOBAL_OFFSET_TABLE_ - ., %ebx")
 
where 1957 gives

    1957:       8b 1c 24                mov    (%esp),%ebx
    195a:       c3                      ret

2. access variables through GOT


GOT for global functions

0. background

Call to global functions becomes
  51:   e8 fc ff ff ff          call   52 <main+0x33>
                        52: R_386_PLT32 strerror

instead of
  38:   e8 fc ff ff ff          call   39 <main+0x27>
                        39: R_386_PC32  strerrror

which will create a PLT entry and call to it.  A relocation to GOT is created,
which is lazy.

000015c4 <sprintf@plt>:
    15c4:       ff a3 10 00 00 00       jmp    *0x10(%ebx)
    15ca:       68 08 00 00 00          push   $0x8
    15cf:       e9 d0 ff ff ff          jmp    15a4 <ZLIB_1.2.0+0x15a4>

1. respective %ebx + 0x10 initializes (lazy!) to 15ca + (library real address - library load address in file)
2. the first time sprintf is called, jump into 15a4, first entry in .plt

000015a4 <__errno_location@plt-0x10>:
    15a4:       ff b3 04 00 00 00       pushl  0x4(%ebx)
    15aa:       ff a3 08 00 00 00       jmp    *0x8(%ebx)

3. %ebx+0x8 points to some function in ld.so, which finds the real address of sprintf (use the relocation) and fill in (%ebx+0x10)