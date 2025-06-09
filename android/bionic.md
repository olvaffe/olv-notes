Android Bionic
==============

## process lifetime

- All executables/libraries are linked with
  `--dynamic-linker=/system/lib/linker`
- When an executable is executed, the kernel will map the dynamic linker as well
  as the executable.  It then jumps to `_start` of the dynamic linker.
  - The dynamic linker is itself statically linked.
  - the kernel passes these data to the linker
    - argc (4 bytes)
    - argv (`argc * 4` bytes)
    - 0 (4 bytes)
    - envvar (dynamic size)
    - 0 (4 bytes)
    - aux data for the execubale (such as PHDR, PHNUM, and ENTRY)
- In dynamic linker's `_start`, `__linker_init` is called
  - it sanitizes the envvar the kernel passes to it
  - it sets up the environment for gdb and debuggerd
  - it allocates an `soinfo` for `argv[0]`
  - it sets up buddy allocator (for loading libraries)
  - `LD_LIBRARY_PATH` is parsed and paths are added to `ldpaths`
  - `LD_PRELOAD` is parsed and libraries are added to `ldpreload_names`
  - `link_image` is called to link the image
  - finally, it returns the address of the entry that `_start` can jump to
    - the entry is `_start` of the executable, which is usually provided by libc
- In executable's `_start`, usually, provided by `crtbegin_dynamic.o`,
  - `__libc_init` is called with the data passed from the kernel as well as
    `jmp main`, which jumps to `main` when called
- When the executable returns from `main` to `__libc_init`, `exit` is called
  - it calls `__cxa_finalize` to call handlers registered with `__cxa_atexit`

## dynamic linking

- `link_image` performs dynamic linking
  - for the execuable, `PT_LOAD` and `PT_DYNAMIC` is first examined
  - `find_library` is called for each `LD_PRELOAD` libraries and then each
    `DT_NEEDED` libraries
  - `PLT_REL`, `REL`, and etc relocation tables are then applied in
    `reloc_library`
  - if fd 0, 1, 2 are not opened, they are pointed to `/dev/null`
- `find_library` locates the location of the library on disk, calls
  `load_library` to map it into the address space, and calls `init_library`,
  which calls `link_image` on the library
- `reloc_library` applies the relocations to an image
  - Given the symbol name, the symbol is first looked up in the image.  If no
    hit, it is searched in preloaded libraries first and then `DT_NEEDED`
    libraries.
    - does this means bionic does not support `RTLD_GLOBAL`?

## dynamic loading

- `libdl.so` defines stub dl functions only used by ld for linking
- the stub dl functions are always overriden by the dynamic linker
  with the real implementations as if the real implementations are always
  preloaded.
  - as a result, `dlsym(RTLD_NEXT, "dlopen")` will never find the real
    implementation (as it is loaded before the caller)
- `dlopen` calls `find_library` and returns `soinfo`
- `dlsym(RTLD_DEFAULT, ...)` looks up the symbol from all `soinfo`s
- `dlsym(RTLD_NEXT, ...)` looks up the symbol from all `soinfo`s loaded after
  the caller
- `dlsym(real_handle, ...)` looks up the symbol from the given `soinfo`

## Linker

- `.text` and entry point is set to `LINKER_TEXT_BASE` 0xb0000100
- entry function `_start` calls `__linker_init`
- `__linker_init` returns program's entry point and `_start` bl to it.
- all symbols are prefixed with `__dl_`

## Workflow

- `solist` begins with `libdl_info`, which is a faked `FLAG_LINKED` `libdl.so`
- `__linker_init` calls `link_image`
- in `link_image`, for each `DT_NEEDED` library, `find_library` is called to
  `load_library` the library and `init_library` (which calls `link_image` and recurses).
  e.g. `/system/bin/ls` uses `liblog.so`, which uses `libc.so`, which uses `libdl.so`
- after `reloc_library` is called in `link_image`, the `si` is marked `FLAG_LINKED`.

## Thread

- In `pthread_create`, a memory to store `struct thread` is malloced.  A stack
  based on anonymous mmap memory is allocated.  The top of the stack is used as
  `tls`.  `clone` is called to fork a new thread.  The parent thread is simple.
  The child thread calls `set_thread_area` to set up gdt/ldt(?) so that `tls` is
  accessible from `%gs:0`.  All threads access their `tls` through `%gs:0` and
  thread-local data is returned.
- `tls[0]` is temporarily used as a mutex until just before `pthread_create`
  returns.  It is used so that the child does not run until the parent unlocks.

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
