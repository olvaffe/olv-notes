Linux binfmt
============

## ELF

- `load_elf_binary` for elf
  - it loops through the executable program headers for `PT_INTERP`
    - if dynamically linked, `interpreter` is the opened file for the dynamic
      loader
  - `begin_new_exec` is the point of no return
    - `de_thread` sends `SIGKILL` to all but the main thread
    - `exec_mmap` transfers `bprm->mm` to `current->mm`
    - `__set_task_comm` sets `tsk->comm` to `bprm->filename`
  - `setup_new_exec`
  - `setup_arg_pages` relocates and grows the user stack
  - it loops through the executable program headers for `PT_LOAD`
    - `elf_load` maps a range
  - `load_elf_interp` loops through the interpreter program headers for
    `PT_LOAD`
    - `elf_load` maps a range
  - `create_elf_tables` inits the user stack
  - `finalize_exec` updates stack size rlimit
  - `START_THREAD` calls `start_thread` to update `pt_regs`
    - on x86, `pt_regs` is at the top of the kernel stack
    - when `ret_from_fork_asm` (pid 1) or `entry_SYSCALL_64` (others) returns
      to userspace,
      - `POP_REGS` pops first part of `pt_regs` to restore userspace regs
      - `iretq` pops second part of `pt_regs` to restore rip, rsp, etc.
      - the cpu returns to elf entrypoint with user stack

## `binfmt_misc`

- it allows userspace to register an interpreter for any binary
  - `mount -t binfmt_misc none /proc/sys/fs/binfmt_misc`
  - `echo <rule> > /proc/sys/fs/binfmt_misc/register`
- since 4.8, `<rule>` can a `fix binary (F)` flag
  - it tells the kernel to open the interpreter immediately when registered
  - this way, when a binary is invoked, the opened interpreter is used
  - without the flag, the interpreter is opened on binary invocation.  It
    can use a different interpreter if chroot
- on debian, `qemu-user-static` registers the rules with the `fix binary` flag
