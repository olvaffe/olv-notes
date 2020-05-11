Task
====

## `struct task_struct`

- outside of boot time setup, a `task_struct` can only be created via
  `_do_fork`
  - `copy_process`
  - `dup_task_struct`
  - `alloc_task_struct_node`
- the callers of `_do_fork` are
  - `kernel_thread`
  - one of the `fork` or `clone` syscalls 

## Boot

- per-cpu `current_task` is statically initialized to statically initialized
  `init_task` defined in `init/init_task.c`
  - `comm` is `INIT_TASK_COMM` (swapper)
  - `active_mm` is `init_mm`
  - `thread_pid` is `init_struct_pid` w/ pid 0
- when `start_kernel` reaches `rest_init`, it spawns two threads
  - `kernel_init` first, which has pid 1
    - it will `execve("/sbin/init")` later
  - `kthreadd` second, which has pid 2
    - kthreadd will respond to future `kthread_create` requests
- `kernel_init`
  - after `workqueue_init`, workqueue queues start running which spawn a
    kthread each
    - `rcu_gp`, `rcu_par_gp`, etc.
  - `init_mm_internals` starts `mm_percpu_wq`
  - after `do_pre_smp_initcalls`, `early_initcall`s are executed.
    - `rcu_spawn_core_kthreads` registers `rcu_cpu_thread_spec`
    - `spawn_ksoftirqd` registers `softirq_threads`
    - `cpu_stop_init` registers `cpu_stop_threads`
    - `idle_inject_init` registers `idle_inject_threads`
    - `rcu_spawn_gp_kthread` starts `rcu_gp_kthread` (comm is `rcu_preempt`)
  - `smp_init`
    - `idle_threads_init` forks an idle thread (`swapper/n`) from `init_task`
      for each CPU.  PIDs are all 0.
    - `cpuhp_threads_init` registers `cpuhp_threads`
    - `bringup_nonboot_cpus` brings secondary CPUs from `CPUHP_OFFLINE` to
      `CPUHP_ONLINE`
      - `cpuhp_up_callbacks` invokes callbacks to bring the CPU up step by
      	step according to `cpuhp_hp_states`
      - `bringup_cpu` enables the CPU core.  On x86, this calls
      	`native_cpu_up`.  The enabled CPU starts from `start_secondary`, with
	task set to the idle task.  After initialization, it calls
	`cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)`
      - the last step calls `sched_cpu_activate` to tell the scheduler the CPU
      	is online
- the idle task on each CPU enters `cpu_startup_entry` and never exits
  - i.e., when it is scheduled on the CPU, the CPU goes idle

## `fork()`

* TBD

## `sys_mmap`

* see `mm`

## ELF

- An ELF file starts with ELF header at file offset 0
  - `e_type`, `ET_DYN` for relocatable and `ET_EXEC` for fixed-address
  - `e_entry`, where the kernel should jump to, or 0
  - `e_phoff`, where the program headers are in the file, or 0
  - `e_shoff`, where the section headers are in the file, or 0
  - `e_ehsize`, ELF header size
  - `e_phentsize` and `e_phnum`, program header size and count
  - `e_shentsize` and `e_shnum`, section header size and count
- A program header describes a segment.  For `PT_LOAD`,
  - `p_offset` is the file offset of the first byte of the segment
  - `p_filesz` is the size of the file image of the segment
  - `p_vaddr` is the virtual address of the segment
  - `p_memsz` is the size of the memory image of the segment
  - `p_flags` is R, W, X
  - `p_align` must be the page size
- A `PT_INTERP` program header specifies the dynamic loader
  - on x86-64, normally the dynamic loader and the executable are both
    relocable.  The kernel loads both at random addresses.
- A section descrbies a section
  - `sh_name`, the name of the section
  - `sh_type`, the type of the section
  - `sh_offset`, the offset of the section in the file
  - `sh_size`, the size of the section in the file
  - `sh_addr`, the address of the section in the memory
* one ELF header followed by
  Program header table, describing zero or more segments
  Section header table, describing zero or more sections
  Data referred to by entries in the program header table, or the section header table
* sections are non-overlapping.  some bytes might not be covered by any section.
* normally, a segment contains one or more sections
* segments are used by kernel for mapping and execution, while sections
  contains data for relocation and linking
* segments of type `PT_LOAD` are to be mapped.  memory size is greater than or
  equal to file size (e.g. bss has no file size)
* e.g. to map two segments
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x08048000 0x08048000 0x4e124 0x4e124 R E 0x1000
  LOAD           0x04e124 0x08097124 0x08097124 0x00730 0x00aa0 RW  0x1000
  there will be two vma (0x08048000 ~ 0x08097000, size 0x4f000)
                        (0x08097000 ~ 0x08098000, size 0x01000)
  a portion of the file is mapped twice.

## `execve()`

* could run an elf executable or script (#!)
* setuid
* If  the  executable  is  a  dynamically linked ELF executable, the
  interpreter named in the `PT_INTERP` segment is used to load the needed shared
  libraries.  This interpreter is typically `/lib/ld-linux.so.1` for binaries
  linked with the Linux libc 5, or `/lib/ld-linux.so.2` for binaries linked with
  the glibc 2.

## `sys_execve`

* `arch/x86/kernel/process_32.c:sys_execve` ->
  `fs/exec.c:do_execve` -> copy argv, envp from userspace, read first
  `BINPRM_BUF_SIZE` bytes, etc.  And calls `search_binary_handler`.
* `load_elf_binary` is called to load the elf.
  the elf header is checked for consistency.
  the program header table is read
  `elf_exec_fileno` is set to be the fd of the executable
  `PT_INTERP` is located and filename of dynamic loader is loaded
  dynamic loader is opened and its elf header is read and checked
  `flush_old_exec` is called to flush old info
  "current" is modified, which is the point of no return
  `arch_pick_mmap_layout` is called to set up mm
  stack is prepared
  segments of type `PT_LOAD` are mapped to the correct locations
  `set_brk` is called
  if no interpreter, `elf_entry` is set to ELF entry address; else, `load_elf_interp`
  `elf_exec_fileno` is closed
  `set_binfmt` is called so that "current" is an ELF executable
  `create_elf_tables` is called to, among others, push argc, argv, envp, and auxp to stack
  finally, `start_thread` is called to set EIP to `elf_entry`
* `load_elf_interp` to load dynamic loader and decide new entry point
  /lib/ld.so specifies 0x0 as its virtual address, which makes itself
  be mapped at around 0xb8000000

## `ld-linux.so`

* <http://www.acsu.buffalo.edu/~charngda/elf.html>
* entry point: `_start` in `glibc/elf/rtld.c`, which calls `_dl_start`
  `_dl_start` returns with program entry point address and `_dl_start_user` jumps to it
  see `sysdeps/i386/dl-machine.h:RTLD_START`
* `_dl_start` -> `_dl_start_final` -> `_dl_sysdep_start` (argc, argv, envp, auxp are popped)
* `_dl_start_user` jumps to the entry point of the program, which is usually
  the beginning of .text section, which is the `_start` function
* see `toolchain`

## Android `linker`

* `Android.mk` makes `.text` to be at `0xB0001000`.  When the linker is
  linked, `ld` finds `_start` (changeable with `-Wl,-entry=<sym>`) and make it
  the entry point.
  * it is built with `-nostdlib` since it has its own `begin.S`
  * it statically links a version of `libc` that has no `malloc`
  * `objcopy --prefix-symbols` is called to rename all symbols to avoid
    collisions with the programs at runtime
* After the control is switched back to the userspace, it starts from `_start`
  * `__linker_init` is called.  It has access to `argc`, `argv`, and `envp`.
    * `LD_*` variables are used to change the behavior of the linker
    * the program is loaded, recursively because of dynamic linking.  The
      entry point of the program is determined and the linker jumps to it.
    * The program may define `.preinit_array`, `.init`, and `.init_array`
      sections.  They are run before jumping to the entry point of the program
* The linker also implements `dl*` API.  While the programes are linked with
  `libdl.so`, it has only stub functions (to help `ld`).  The implementation
  is inside the linker.
* The linker defines `rtld_db_dlactivity` and `_r_debug`, which can be used
  to help `gdb`

## Android `bionic`

* All executables are built with `-nostdlib` and bionic's `crt*`
* `crtbegin_dynamic.S` calls `__libc_init` to initialize bionic and 
  call program's `main`
  * it also defines `.preinit_array` and `.init_array` to be called by the
    linker.  Specifically, `__libc_preinit` will be called
* after program's `main` returns, `exit` is called wit the return value.
  * it calls all functions registered with `atexit()`
  * it terminates the process by calling `_exit()`
* `__libc_preinit`
  * it prepares the stack and for the threads
  * it initializes the TLS area
  * it calls `__system_properties_init` to initialize properties
    * `/init` prepares the storage for `__system_property_area__` and set
      `ANDROID_PROPERTY_WORKSPACE`.  All processes spawned by `/init` will use
      the same storage by mapping `ANDROID_PROPERTY_WORKSPACE`.
