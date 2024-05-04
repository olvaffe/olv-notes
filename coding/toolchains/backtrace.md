Backtrace
=========

## Proejcts

- gcc and llvm both provide an unwinder
  - <https://gcc.gnu.org/git/?p=gcc.git;a=tree;f=libgcc;hb=HEAD>
  - <https://github.com/llvm/llvm-project/tree/main/libunwind>
  - `unwind.h` is the header
    - Itanium C++ ABI: Exception Handling Level I. Base ABI
      - <https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html>
      - `_Unwind_RaiseException`
      - `_Unwind_Resume`
      - `_Unwind_DeleteException`
      - `_Unwind_GetGR`
      - `_Unwind_SetGR`
      - `_Unwind_GetIP`
      - `_Unwind_SetIP`
      - `_Unwind_GetRegionStart`
      - `_Unwind_GetLanguageSpecificData`
      - `_Unwind_ForcedUnwind`
    - gcc extension that llvm also implements
      - `_Unwind_Backtrace`
      - `_Unwind_FindEnclosingFunction`
      - `_Unwind_GetBSP`
      - `_Unwind_GetCFA`
      - `_Unwind_GetDataRelBase`
      - `_Unwind_GetIPInfo`
      - `_Unwind_GetTextRelBase`
      - `_Unwind_Resume_or_Rethrow`
  - `_Unwind_Backtrace` back traces the stack frames
    - the caller passes in a callback that is invoked at each frame
    - the caller uses `_Unwind_GetIPInfo` and `_Unwind_GetBSP` to get the ip
      and bsp values of the frame
  - there is no symbol resolve
- `libunwind`
  - <https://www.nongnu.org/libunwind/>
  - <https://git.savannah.gnu.org/gitweb/?p=libunwind.git;a=summary>
  - `libunwind.h` is the header
  - usage
    - `unw_getcontext` to get the regs of the local process
    - `unw_init_local` to initialize
    - `while (unw_step(&cursor))` to step through the stack frames
      - `unw_get_reg` to get ip and sp
      - `unw_get_proc_name` to get the symbol name
        - no dwarf support?
  - it can unwind a local process or a remote process
  - it also provides the c++ exception handling abi
    - that is, the same `unwind.h` itnerface that gcc and llvm provide
- `libbacktrace`
  - <https://gcc.gnu.org/git/?p=gcc.git;a=tree;f=libbacktrace;hb=HEAD>
    - it is a part of gcc that gcc devs use for gcc debugging
    - it depends on `unwind.h` which is also provided by gcc
  - <https://github.com/ianlancetaylor/libbacktrace> is an indendent version
    - since `unwind.h` is provided by gcc and llvm, it works
  - `backtrace.h` is the header
  - `backtrace_create_state` creates a `backtrace_state`
  - `backtrace_full` gets a full stack backtrace
    - it calls `_Unwind_Backtrace` internally
    - it calls `backtrace_pcinfo` to resolve the addr to file name, line
      number, and symbol name

## Android

- <https://android.googlesource.com/platform/external/libunwind/+/refs/heads/main/>
  - this has been removed since android 12
  - <https://www.nongnu.org/libunwind/> is the upstream
- <https://android.googlesource.com/platform/system/unwinding/+/refs/heads/android13-dev/libbacktrace/>
  - this has been removed since android 14
  - `Backtrace.h` is the header
  - `Backtrace::Create` creates a `class Backtrace`
    - it creates an `UnwindStackCurrent` or `UnwindStackPtrace` internally
    - `BacktraceCurrent::Unwind` calls `UnwindStackCurrent::UnwindFromContext`
      which calls `Backtrace::Unwind`
    - `UnwindStackPtrace::Unwind` calls `Backtrace::Unwind`
  - `Backtrace::Unwind` is a wrapper to `libunwindstack`
    - defines a local `unwindstack::Unwinder`
    - `unwinder.Unwind()` to unwind
    - `unwinder.frames()` to get the frames
- <https://android.googlesource.com/platform/system/unwinding/+/refs/heads/main/libunwindstack/>
  - `unwindstack/Unwinder.h` is the header
  - `unwindstack::ThreadUnwinder` unwinds for the current thread
    - it is a subclass of `unwindstack::UnwinderFromPid`
    - it uses `getpid()` as the pid
  - `unwindstack::UnwinderFromPid` unwinds for the given pid
    - it is a subclass of `unwindstack::Unwinder`
    - it creates a `Maps` and a `Memory` for the pid and calls down to
      `unwindstack::Unwinder::Unwind`
    - `Maps::Parse` parses `/proc/self/maps` or `/proc/<pid>/maps`
    - `Memory::Read` reads the pid memory
      - `ProcessVmRead` calls `process_vm_readv`
  - the caller is responsible for creating `Regs` before unwinding
    - `unwindstack::Regs::CreateFromLocal` creates `Regs` for the current
      thread
      - `unwindstack::RegsGetLocal` must be called to get the reg values
    - `unwindstack::Regs::CreateFromUcontext` creates `Regs` for another
      context
      - it reads the reg values from the context
    - `unwindstack::Unwinder::SetRegs` passes the reg to the unwinder
  - `unwindstack::Unwinder::Unwind` unwinds
    - `maps_->Find(regs_->pc())` finds a `MapInfo`
    - `map_info->GetElf(process_memory_, arch_)` returns a `Elf`
    - `FillInFrame` fills in a `FrameData`
    - `elf->Step` steps to the next frame
      - it tries `.debug_frame`
      - else, it tries `.eh_frame_hdr` or `.eh_frame`
      - else, it tries `.gnu_debugdata`
      - else, it fails
      - it does not rely on frame pointer
    - `elf->GetFunctionName` resolves the pc to a symbol and an offset
      - it tries `SHT_SYMTAB` and `SHT_DYNSYM`
      - else, it tries `.gnu_debugdata`
      - else, it fails
  - putting everything together, here is how the unwinder works for a local
    process
    - `RegsGetLocal` gets the current regs
    - `Maps::Find` uses `/proc/self/maps` to find the elf that is mapped to
      the current pc
    - `MapInfo::GetElf` uses `process_vm_readv` to read and parse interesting
      elf sections
    - `Elf::Step` uses the debug data to figure the previous frame and update
      regs
    - `Elf::GetFunctionName` uses the symbol tables to resolve `pc` to the
      symbol name and offset
