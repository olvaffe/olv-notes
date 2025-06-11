Kernel exec
===========

## `kernel_execve`

- boot
  - pid 0 calls `user_mode_thread` from `rest_init` to fork pid 1
    - on x86, `copy_thread` sets the return addr to `ret_from_fork_asm`
      - pid 1 `ret_from_fork_asm` calls `ret_from_fork`, which calls
        `kernel_init` before returning
  - pid 1 calls `kernel_execve` from `kernel_init` to exec `/init` or
    `/sbin/init`
- `getname_kernel` allocs `filename` and copies `/init` or `/sbin/init` to it
- `alloc_bprm` allocs a `linux_binprm`
  - `do_open_execat` opens the file for exec
  - `bprm_mm_init` allocs a `mm_struct`
    - `bprm->rlim_stack` is copied from `current->signal->rlim[RLIMIT_STACK]`
    - `bprm->vma` is the initial stack
      - `vma_end` is `STACK_TOP_MAX` (top of the addr space)
      - `vma_start` is one page below
      - flags has `VM_STACK_FLAGS` which includes `VM_GROWSDOWN`
    - `bprm->p` points to `vma->vm_end - sizeof(void *)`
- `count_strings_kernel` inits `bprm->argc` and `bprm->envc`
- `bprm_stack_limits` calculates stack size limit
  - `limit` is typically 2MB
    - `_STK_LIM` is 8MB
    - `bprm->rlim_stack.rlim_cur` is typically 8MB
  - `bprm->argmin` is `bprm->p - limit`, which caps how much the stack can
    grow
    - `bprm_hit_stack_limit` returns true when `bprm->p` is below
      `bprm->argmin`
- `copy_string_kernel` copies filename to stack
- `copy_strings_kernel` copies `envp` and `argv` to stack
- `bprm_execve`
  - `prepare_bprm_creds` creates `bprm->cred`, inheriting from `current`
  - `sched_exec` picks the cpu?
  - `exec_binprm` loads the file and updates regs
    - `search_binary_handler` invokes binfmt `load_binary`, such as
      `load_elf_binary`

## Process User Stack

- `bprm_mm_init` mmaps user stack
  - `bprm->mm` is the task `mm_struct`
  - `bprm->vma` is the stack vma
    - `vm_end` is `STACK_TOP_MAX` (`0x7ffffffff000` on x86)
    - `vm_start` is 1 page below (`0x7fffffffe000` on x86)
    - flags contains `VM_GROWSDOWN`
  - `bprm->p` points to `vma->vm_end - sizeof(void *)`
- `bprm_stack_limits` sets string pool limit
  - `limit` is typically capped by `bprm->rlim_stack.rlim_cur / 4` (2MB)
  - there will also be `bprm->argc` plus `bprm->envc` pointers
  - `bprm->argmin = bprm->p - limit`
- first `copy_string_kernel` copies `bprm->filename` to the stack
  - this copies to top of stack
  - `bprm->p` is decremented
- first `copy_strings` copies `bprm->envp` to the stack
- second `copy_strings` copies `bprm->argv` to the stack
- `setup_arg_pages`
- `create_elf_tables`
  - it reserves space from the stack and add values from bottom to top
    - `bprm->argc` for argc
    - `bprm->argc` pointers to `bprm->argv` string pool
    - 0 to delimit argv
    - `bprm->envc` pointers to `bprm->envp` string pool
    - 0 to delimit envp
    - auxv
  - on x86, the abi is documented in `System V Application Binary Interface`
    - `3.4 Process Initialization`

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
