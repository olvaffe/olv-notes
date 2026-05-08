# Kernel and Stack Trace

## Arch-Specifics

- `arch_stack_walk` is the unwinder
  - if `task` is `current`,
    - if `regs`, it unwinds the interrupted context
      - e.g., an exception handler walks the stack of a faulty function
    - else, it unwinds the caller
  - else, it assumes the task is sleeping and unwinds from its
    `context_switch`
- `show_stack` dumps the stack of the current or the specified task
  - it unwinds the current or the specified task, usually using the unwinder
  - it uses `%pSb` to print symbolized addrs
- `show_regs`
  - it dumps the saved regs
  - it unwinds the interrupted context, usually using the unwinder

## `CONFIG_STACKTRACE`

- `stack_trace_save` calls `arch_stack_walk` to save the caller's stack to an
  caller-provided addr array, for later use
- `stack_trace_print` uses `%pS` to print symboalized addrs of the addr array

## `CONFIG_STACKDEPOT`

- `stack_depot_save` saves a stack addr array to a global depot
  - the global depot is designed to handle millions of addr arrays

## `CONFIG_KALLSYMS`

- `symbol_string` handles `%pS` with the help of `__sprint_symbol`
- `kallsyms_lookup`
  - `is_ksym_addr` tests if the addr is in kernel
    - it binary searches kallsyms
  - else, `module_address_lookup` searches in modules
