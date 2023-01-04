Kernel exec
===========

## `execve()`

- could run an elf executable or script (#!)
- setuid
- If  the  executable  is  a  dynamically linked ELF executable, the
  interpreter named in the `PT_INTERP` segment is used to load the needed shared
  libraries.  This interpreter is typically `/lib/ld-linux.so.1` for binaries
  linked with the Linux libc 5, or `/lib/ld-linux.so.2` for binaries linked with
  the glibc 2.

## `sys_execve`

- `arch/x86/kernel/process_32.c:sys_execve` ->
  `fs/exec.c:do_execve` -> copy argv, envp from userspace, read first
  `BINPRM_BUF_SIZE` bytes, etc.  And calls `search_binary_handler`.
  - `bprm_mm_init` sets up mm and the stack
    - the stack is a vma ending at `STACK_TOP_MAX` and with `VM_GROWSDOWN`
    - on page fault, `expand_stack` is called to grow the vma down
- `load_elf_binary` is called to load the elf.
  the elf header is checked for consistency.
  the program header table is read
  `elf_exec_fileno` is set to be the fd of the executable
  `PT_INTERP` is located and filename of dynamic loader is loaded
  dynamic loader is opened and its elf header is read and checked
  `flush_old_exec` is called to flush old info
  "current" is modified, which is the point of no return
- `arch_pick_mmap_layout` is called to set up mm
- `setup_arg_pages`
  - stack is prepared at `STACK_TOP`
    - `STACK_TOP` is (`1<<47 - PAGE_SIZE)` on x86-64
- segments of type `PT_LOAD` are mapped to the correct locations
- `set_brk` is called
- if no interpreter, `elf_entry` is set to ELF entry address; else, `load_elf_interp`
- `elf_exec_fileno` is closed
- `set_binfmt` is called so that "current" is an ELF executable
- `create_elf_tables` is called to, among others, push argc, argv, envp, and auxp to stack
- finally, `start_thread` is called to set EIP to `elf_entry`
- `load_elf_interp` to load dynamic loader and decide new entry point
  /lib/ld.so specifies 0x0 as its virtual address, which makes itself
  be mapped at around 0xb8000000

## `ld-linux.so`

- <http://www.acsu.buffalo.edu/~charngda/elf.html>
- entry point: `_start` in `glibc/elf/rtld.c`, which calls `_dl_start`
  `_dl_start` returns with program entry point address and `_dl_start_user` jumps to it
  see `sysdeps/i386/dl-machine.h:RTLD_START`
- `_dl_start` -> `_dl_start_final` -> `_dl_sysdep_start` (argc, argv, envp, auxp are popped)
- `_dl_start_user` jumps to the entry point of the program, which is usually
  the beginning of .text section, which is the `_start` function
- see `toolchain`

## Android `linker`

- `Android.mk` makes `.text` to be at `0xB0001000`.  When the linker is
  linked, `ld` finds `_start` (changeable with `-Wl,-entry=<sym>`) and make it
  the entry point.
  - it is built with `-nostdlib` since it has its own `begin.S`
  - it statically links a version of `libc` that has no `malloc`
  - `objcopy --prefix-symbols` is called to rename all symbols to avoid
    collisions with the programs at runtime
- After the control is switched back to the userspace, it starts from `_start`
  - `__linker_init` is called.  It has access to `argc`, `argv`, and `envp`.
    - `LD_*` variables are used to change the behavior of the linker
    - the program is loaded, recursively because of dynamic linking.  The
      entry point of the program is determined and the linker jumps to it.
    - The program may define `.preinit_array`, `.init`, and `.init_array`
      sections.  They are run before jumping to the entry point of the program
- The linker also implements `dl*` API.  While the programes are linked with
  `libdl.so`, it has only stub functions (to help `ld`).  The implementation
  is inside the linker.
- The linker defines `rtld_db_dlactivity` and `_r_debug`, which can be used
  to help `gdb`

## Android `bionic`

- All executables are built with `-nostdlib` and bionic's `crt*`
- `crtbegin_dynamic.S` calls `__libc_init` to initialize bionic and 
  call program's `main`
  - it also defines `.preinit_array` and `.init_array` to be called by the
    linker.  Specifically, `__libc_preinit` will be called
- after program's `main` returns, `exit` is called wit the return value.
  - it calls all functions registered with `atexit()`
  - it terminates the process by calling `_exit()`
- `__libc_preinit`
  - it prepares the stack and for the threads
  - it initializes the TLS area
  - it calls `__system_properties_init` to initialize properties
    - `/init` prepares the storage for `__system_property_area__` and set
      `ANDROID_PROPERTY_WORKSPACE`.  All processes spawned by `/init` will use
      the same storage by mapping `ANDROID_PROPERTY_WORKSPACE`.
