GDB
===

## `gdb`
- `gdb <prog>`, `gdb <prog> <pid`, or `gdb <prog> <core>`

## Symbols

- `strip`
  - `-g` removes debug symbols (in `.debug_*`)
  - `-x` additionally removes local symbols (in `.symtab`)
  - `-s` removes `.symtab` completely
- gdb can build symbol table from various fomrats
  - DWARF, aka debug symbols
  - `.symtab` symbol table
  - `.dynsym` dynamic symbol table
- <https://sourceware.org/gdb/current/onlinedocs/gdb/Files.html>
  - `set sysroot` and `show sysroot`
    - `solib-absolute-prefix` is an alias for `sysroot`
  - `set solib-search-path` and `show solib-search-path`
    - additionaly paths to search when `sysroot` fails
- <https://sourceware.org/gdb/current/onlinedocs/gdb/Separate-Debug-Files.html>
  - `set debug-file-directory` and `show debug-file-directory`

## Source Path

- DWARF contains source file info
  - `DW_TAG_compile_unit` has
    - `DW_AT_comp_dir("/home/olv/projects/vkcube/out")`
    - `DW_AT_name("../main.c")`
  - various symbols have
    - `DW_AT_decl_file("/home/olv/projects/vkcube/out/../main.c")`
- <https://sourceware.org/gdb/current/onlinedocs/gdb/Source-Path.html>
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
- `set solib-absolute-prefix <sysroot>`
- `set debug-file-directory <debug-file-dir>`
- `set solib-search-path <relative-paths>`
- `set directories <source-file-dirs>`

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
