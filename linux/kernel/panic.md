Kernel Panic
============

## `WARN` and `BUG`

- archs typically define
  - `CONFIG_BUG`
  - `CONFIG_GENERIC_BUG`
  - `HAVE_ARCH_BUG`
  - `__WARN_FLAGS`
- upon generic `WARN`,
  - `__warn_printk` prints
    - `------------[ cut here ]------------`
    - `<error message>`
  - `__WARN_FLAGS` is arch-specific
    - x86 expands to `_BUG_FLAGS` and is handled similar to `BUG`
    - arm expands to `__BUG_FLAGS` and is handled similar to `BUG`
- upon arch-specific `BUG`,
  - x86 epands to `_BUG_FLAGS`
    - `_BUG_FLAGS_ASM` records a `bug_entry` in `__bug_table` section and
      execs `ASM_UD2` instr to trap `DEFINE_IDTENTRY_RAW(exc_invalid_op)`
    - `handle_bug` calls generic `report_bug`
    - if `BUG`, `handle_invalid_op` calls `do_error_trap` and `do_trap` calls
      `die`
  - arm expands to `__BUG_FLAGS`
    - `ASM_BUG_FLAGS` records a `bug_entry` in `__bug_table` and execs `brk`
      instr to trap `el1h_64_sync_handler`
    - `bug_brk_handler` calls generic `report_bug`
    - if `BUG`, it also calls `die`
- generic `report_bug`
  - `find_bug` finds the recorded `bug_entry`
  - if `WARN`, it calls generic `__warn` and returns `BUG_TRAP_TYPE_WARN`
    - `cut here` has been printed by `__warn_printk`
  - if `BUG`, it
    - prints `------------[ cut here ]------------`
    - prints `kernel BUG at ...`
    - returns `BUG_TRAP_TYPE_BUG`
- `WARN` stops here
- `BUG` goes on to call `die`, aka oops
- x86 `die`
  - `oops_begin`
    - generic `oops_enter`
  - `__die_header` prints `Oops: ...`
  - `__die_body`
    - `show_regs` prints regs
    - `print_modules` prints modules
  - `oops_end`
    - generic `oops_exit`
      - `print_oops_end_marker` prints `---[ end trace %016llx ]---`
      - `kmsg_dump` dumps the oops to pstore, etc.
        - subsys calls `kmsg_dump_register` to register a dumper
    - `__show_regs` dumps regs again
    - depending on severity and config, `panic` is called
- arm `die`
  - generic `oops_enter`
  - `__die`
    - prints `Internal error: Oops - BUG ...`
    - `print_modules` prints modules
    - `show_regs` prints regs
    - `dump_kernel_instr` prints `Code: ...`
  - generic `oops_exit`
  - depending on severity and config, `panic` is called

## `panic`

- an oops (e.g., `BUG()`) may trigger `panic`
  - `CONFIG_PANIC_ON_OOPS` or too severe
- generic `panic`
  - prints `Kernel panic - not syncing: ...`
  - `kmsg_dump_desc` records the panic to pstore, drm, etc.
  - `dump_stack` prints call stack
  - prints `---[ end Kernel panic - not syncing: %s ]---`
  - infinite loop unless `CONFIG_PANIC_TIMEOUT`
