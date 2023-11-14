GDB
===

## `gdb`

- `gdb <prog>`, `gdb <prog> <pid>`, or `gdb <prog> <core>`

## Debugging with GDB

- 2 Getting In and Out of GDB
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Invocation.html>
  - choosing files
    - `-s` is `symbol-file`
    - `-e` is `exec-file`
    - `-se` is `file`
    - `-c` is `core-file`
    - `-p` is `attach`
  - executing commands
    - `-eix` and `-eiex` for earlyinit commands
    - `-ix` and `-iex` for init commands
    - `-x` and `-ex` for commands (after file loading)
  - init files
    - `-nx` to skip all
    - `-nh` to skip those in home dir
  - `--args` stops arg parsing for gdb itself
  - `show logging`
    - save gdb output to a file
- 3 GDB Commands
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Commands.html>
  - `help`
  - `apropos` to search through all commands
  - `info` to show the state of the program
  - `set`
  - `show` to show the state of gdb
- 4 Running Programs Under GDB
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Running.html>
  - terminology
    - `program` is the binary to be debugged
    - `target` is the environment `program` executes in
    - `inferior` is the state of a program execution
      - usually a process, if `target` supports processes
  - inferiors
    - when gdb starts, there is 1 `inferior`
      - `info inferiors`
    - an inferior represents the state of a program
      - `file` sets the program of the current inferior
      - `target native` connects the current inferior to the native target
        - because `show auto-connect-native-target` is on by default, this is
          optional
      - `run` starts the program in the native target
    - not all targets support or require `run`
      - `target core`
      - `target remote`
      - `run` might appear to work only because of
        `auto-connect-native-target`
    - more inferiors can be added with `add-inferior`
  - starting
    - `run`
    - `start`
    - `starti`
    - `show args`
  - `attach` and `detach`
    - `/proc/sys/kernel/yama/ptrace_scope`
  - inferiors
    - `info inferiors`
    - `info connections`
    - `inferior`
  - threads
    - `info threads`
    - `thread`
    - `thread apply all`
    - `taas` is `thread apply all -s`
  - forking
    - `show follow-fork-mode`
    - `show detach-on-fork`
    - `show follow-exec-mode`
- 5 Stopping and Continuing
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Stopping.html>
  - breakpoints
  - watchpoints
  - catchpoints
    - `catch exec`
    - `catch syscall`
    - `catch fork`
    - `catch load` and `catch unload`
    - `catch signal`
  - signals
    - `info signals`
    - `handle`
- 8 Examining the Stack
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Stack.html>
  - frame
    - `info frame`
    - `info args`
    - `info locals`
    - `frame apply all`
    - `faas` is `frame apply all -s`
- 9 Examining Source Files
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Source.html>
- 10 Examining Data
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Data.html>
  - `display`
  - `show print`
  - `show values`
  - `info registers`
  - `info auxv`
  - `info os`
  - `gcore`
- 18 GDB Files
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/GDB-Files.html>
  - program
    - `file` is `exec-file` and `symbol-file`
    - `exec-file` is `target exec`
    - `symbol-file` loads program symbols from the specified file
      - this clears the current symbols first
    - `core-file` is `target core`
    - `add-symbol-file` adds program symbols from the specified file
      - this is intended to be used when the program can dynamically load code
        through means other than dlopen
    - `info files`
  - shared libraries
    - `show auto-solib-add` auto-loads symbols for shared libraries
    - `info sharedlibrary`
    - `sharedlibrary` and `nosharedlibrary` to force load/unload symbols for
      shared libraries
  - remote debug
    - `set sysroot`
    - `set solib-search-path path`
    - `show debug-file-directory`
  - `show index-cache` saves symbol index cache to speed up symbol loading
    next time
- 19 Specifying a Debugging Target
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Targets.html>
  - `target exec`
  - `target core`
  - `target remote`
  - `target native`
- 20 Debugging Remote Programs
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Remote-Debugging.html>
  - `target remote`
  - `target extended-remote`
  - `gdbserver`
- 21 Configuration-Specific Information
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Configurations.html>
  - `info proc all` shows various info under `/proc/<pid>`
- 22 Controlling GDB
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Controlling-GDB.html>
  - command history
    - `show history` saves command history to a file
    - `show commands` shows command history of the current session, including
      those loaded from the history file
  - `show pagination` whether to disable paging
  - `show confirm` whether to ask for confirmation
  - `show verbose` be verbose about what gdb is doing internally
  - `show debug` be more verbose about what gdb is doing internally
- 23 Extending GDB
  - <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Extending-GDB.html>
  - `source` reads gdb commands from the specified file
  - `alias` defines a shortcut

## Symbols

- `strip`
  - `-g` removes debug symbols (in `.debug_*`)
  - `-x` additionally removes local symbols (in `.symtab`)
  - `-s` removes `.symtab` completely
- gdb can build symbol table from various fomrats
  - DWARF, aka debug symbols
  - `.symtab` symbol table
  - `.dynsym` dynamic symbol table
- <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Files.html>
  - `set sysroot` and `show sysroot`
    - `solib-absolute-prefix` is an alias for `sysroot`
  - `set solib-search-path` and `show solib-search-path`
    - additionaly paths to search when `sysroot` fails
- <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Separate-Debug-Files.html>
  - `set debug-file-directory` and `show debug-file-directory`
- experiment with `set debug separate-debug-file on`,
  - `$sysroot` is expanded to the value of `sysroot`
  - `$debugdir` is expanded to the value of `debug-file-directory`
  - assume gdb finds `$sysroot/usr/lib/libfoo.so.1` for `libfoo.so.1`
  - gdb looks for symbols in `$sysroot/usr/lib/libfoo.so.1` itself
  - if no symbols, gdb looks for symbols using the build-id
    - the build id is given by the `.note.gnu.build-id` section
    - `$debugdir/.build-id/<hash1>/<hash2>.debug`
    - `$sysroot/$debugdir/.build-id/<hash1>/<hash2>.debug`
  - if still no symbols, gdb looks for symbols using the debug link
    - the debug link is given by the `.gnu_debuglink` section
      - `readelf -wk $sysroot/usr/lib/libfoo.so.1` dumps the section, which
        gives a name such as `libfoo.so.1.2.3.debug`
    - `$sysroot/usr/lib/libfoo.so.1.2.3.debug`
    - `$sysroot/usr/lib/.debug/libfoo.so.1.2.3.debug`
    - `$debugdir/$sysroot/usr/lib/libfoo.so.1.2.3.debug`
    - `$debugdir/usr/lib/libfoo.so.1.2.3.debug`
    - `$sysroot/$debugdir/usr/lib/libfoo.so.1.2.3.debug`

## Source Path

- DWARF contains source file info
  - `DW_TAG_compile_unit` has
    - `DW_AT_comp_dir("/home/olv/projects/vkcube/out")`
    - `DW_AT_name("../main.c")`
  - various symbols have
    - `DW_AT_decl_file("/home/olv/projects/vkcube/out/../main.c")`
- <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Source-Path.html>
  - `set directories` and `show directories`
    - by default, `$cdir:$cwd`
      - `$cdir` is `DW_AT_comp_dir`
      - `$cwd` is the current directory
    - when gdb searches for `/path/to/main.c`, it searches
      - `/path/to/main.c`
      - `/path/to/main.c` under each of `directories`
      - `$cdir/path/to/main.c` under each of `directories`
      - `main.c` under each of `directories`

## Debug gdb itself

- `help set debug`

## Breakpoints

- gdb uses software breakpoints on x86-64 and aarch64
  - gdbserver uses hardware breakpoints instead
- software breakpoints work by instruction patching
  - it repalces the instruction at the target address by a special instruction
  - it works by opening `/proc/<pid>/task/<tid>/mem` for raw memory access
  - does not work on cros because raw memory access is read-only
    <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/7c10b00c80d415128d54245d10255e883c62fa4a>

## `gdbserver`

- to run a program under `gdbserver`, `gdbserver :1234 prog args`
- to attach, `gdbserver --attach :1234 pid`
  - use `ps -AL` to find the tid, for a multi-threaded program

## cross/remote debugging

- `gdb -ex "target remote :1234" prog`
- `set sysroot <sysroot>`
  - the old name is `set solib-absolute-prefix <sysroot>`
- `set debug-file-directory <debug-file-dir>`
- `set solib-search-path <relative-paths>`
- `set directories <source-file-dirs>`
- remote debugging tips
  - gdb defaults `sysroot` to `target:`
    - this retrieves the executable and the shared libraries from the target,
      which is the remote when remote debugging
    - `set sysroot /` to use the local files even if they might be incorrect
      - or `set sysroot /not-exist`
  - gdb defaults `directories` to `$cdir:$cwd`
    - this finds source files relative to `$cdir` or `$cwd`
    - if they are not enough, `set directories <out>`
  - gdb defaults `solib-search-path` to `.`
    - if they are not enough, `set solib-search-path <out>`

## Threading

- `gdb` and `gdbserver` loads `libthread_db.so` at runtime to enable
  debugging of multithreaded applications

## DWARF

- DWARF uses a tree of DIEs to describe a program
  - a DIE, debugging information entry, consists of a type (`DW_TAG_*`) and
    some attributes (`DW_AT_*`)
  - a DIE can have a parent DIE, sibling DIEs, and children DIEs
- common DIEs
  - `DW_TAG_compile_unit` is a compilation unit
  - `DW_TAG_subprogram` is a function
  - `DW_TAG_variable` is a variable
  - `DW_TAG_base_type` is a base type like a `unsigned int`
  - `DW_TAG_const_type` is a const type like a `const int`
  - `DW_TAG_pointer_type` is a pointer type like a `int *`
  - `DW_TAG_typedef` is a typedef like a `typedef int foo`
  - `DW_TAG_array_type` is an array type like a `int[8]`
    - there is a child `DW_TAG_subrange_type` to describe the array size
  - `DW_TAG_structure_type` and `DW_TAG_union_type` are a struct or union type
    like a `struct Foo { ... }`
    - there are `DW_TAG_member` children for the struct members
  - `DW_TAG_enumeration_type` is an enum type like a `enum Foo { ... }`
    - there are `DW_TAG_enumerator` children for the enum values
  - `DW_TAG_subroutine_type` is a function type like a `int (*)(int)`
    - there are `DW_TAG_formal_parameter` children for function parameters
  - `DW_TAG_lexical_block` is a lexical scope like a `{ ... }`
  - `DW_TAG_unspecified_parameters` is a variable argument list (`void foo(...)`)
- `dwarfdump`
  - `-b` dumps `.debug_abbrev`
    - abbreviations used by `.debug_info`
  - `-r` dumps `.debug_aranges`
    - lookup table to map addresses to compilation units
  - `-i` dumps `.debug_info`
    - the core DWARF info
  - `-l` dumps `.debug_line`
    - line number information
    - map addresses to line numbers?
  - `-s` dumps `.debug_str`
    - string table used by `.debug_info`
  - `-b` dumps `.debug_line_str`
  - `--print-raw-rnglists` dumps `.debug_rnglists`
  - `--print-raw-loclists` dumps `.debug_loclists`
  - there is also `llvm-dwarfdump` that can pretty print `.debug_info`
  - there is also `readelf --debug-dump`

## debuginfod

- `DEBUGINFOD_URLS`
