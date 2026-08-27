# Wine

## Releases

- Wine 1.0
  - 2008
- Wine 1.2
  - 2010
  - most of D3D9 supported
  - D3D10 started
- Wine 1.4
  - 2012
  - more complete D3D9 support
- Wine 1.6
  - 2013
  - native mac os x driver; x11 driver no longer used on mac
  - improvements to D3D9 and D3D10
- Wine 1.8
  - 2015
  - Direct2D support
  - DXGI support
  - early D3D10 and D3D11 support
- Wine 2.0
  - 2017
  - more D3D10 and D3D11 features
- Wine 3.0
  - 2018
  - more D3D10 and D3D11 features
  - Direct3D multi-threaded command stream support
  - native Android driver
- Wine 4.0
  - 2019
  - initial D3D12 support, depending on vkd3d
  - multi-threaded command stream enabled by default
  - more D3D10 and D3D11 features
  - vulkan backend for `winex11.drv` and `winemac.drv`
- Wine 5.0
  - 2020
  - most builtin modules have been converted from `.dll.so` to `.dll`
    - when MinGW-w64 cross-compiler is used
    - ongoing process
- Wine 6.0
  - 2021
  - core modules are converted to `.dll`
    - NTDLL, KERNEL32, GDI32, USER32, etc
  - an mechanism to call UNIX library from `.dll`
    - `foo.dll` can have a UNIX `foo.so` counterpart
  - `libwine.so` is deprecated
  - experimental vulkan backend for wined3d
    - i.e., `d3d->vk` instead of `d3d->gl`
- Wine 7.0
  - 2022
  - most modules are converted to `.dll`
  - new WoW64 architecture
- Wine 8.0
  - 2023
  - all modules are converted to `.dll`
  - most modules use NT syscall to transition from PE to UNIX
- Wine 9.0
  - 2024
  - all modules use NT syscall to transition from PE to UNIX, completing new
    WoW64 support which allows 32-bit PE code running in 64-bit Unix process
- Wine 10.0
  - 2025
  - full ARM64(EC) support
  - wayland support
- Wine 11.0
  - 2026
  - new WoW64 mode
  - ntsync

## Envvars

- `WINEPREFIX` defaults to a shared prefix, `~/.wine`
  - `wineboot` bootstraps a prefix
    - `drive_c` is drive C
    - `system.reg` is `HKLM`
    - `user.reg` is `HKCU`
- `WINEDLLOVERRIDES` overrides loader behavior
  - `dll1,dll2=mode1;dll3=mode2`
  - `mode` can be
    - `n`: native, searched in work dir and system dir
    - `b`: builtin, from wine
    - `n,b` or `b,n`: with fallback
    - empty: always missing
- `WINEDEBUG` controls logging
  - `+seh` logs structured exception handling (crashes)
  - `+debugstr` logs debug strs from executables
  - `+loaddll` logs dll loading
  - `-all,err+all` disables for all channels then enables err for all channels
  - `+timestamp,+pid,+tid,+threadname` enables all 4 metadata channels
    - prefix each log by timestamp, pid, tid, and threadname

## Initialization Process

- <https://wiki.winehq.org/Wine_Developer%27s_Guide/Kernel_modules>
- when user invokes "wine foo.exe"
  - entrypoint is `main` in `loader/main.c`
  - `realpath("/proc/self/exe", NULL)` to get the bindir
  - `dlopen(BIN_TO_DLLDIR "/ntdll.so", RTLD_NOW)` to open `ntdll.so`
  - calls `__wine_main` in `ntdll.so`
- `__wine_main`
  - `init_paths`
    - `dll_dir` is determined from `dladdr` and `realpath`
    - `bin_dir` is determined from `realpath("/proc/self/exe", NULL)`
    - `dll_paths` is set to `dll_dir` and `$WINEDLLPATH`
    - `home_dir` is set to `$HOME`
    - `home_dir` is set to `$HOME`
    - `user_name` is set to `$USER`
    - `config_dir` is set to `$WINEPREFIX` or `$HOME/.wine`
  - because `$WINELOADERNOEXEC` is not set, `loader_exec` is called
    - this `execve` with `argv[0]` being `wine-preloader`and the rest copied
      from the current invocation
    - `$WINELOADERNOEXEC=1` is set
- `wine-preloader`
  - statically linked with `-static -nostartfiles -nodefaultlibs`
  - entrypoint is `_start` which calls `wld_start`
  - `wld_start` simulates kernel elf loader to load the binary (`wine`) and
    the elf interpreter (`/lib/ld-linux.so.2`)
  - `preload_info` is pre-filled with low 64k, dos area, low memory area, and
    stack and heap
    - each range is mmaped with `PROT_NONE` and
      `MAP_FIXED | MAP_PRIVATE | MAP_ANON | MAP_NORESERVE`
  - make the last page exec, `mprotect(0x7ffff000, 4096, PROT_EXEC | PROT_READ)`
  - load the binary (`wine`) and the interpreter (`/lib/ld-linux.so.2`)
  - because `wine` has `wine_main_preload_info`, it is set to `preload_info`
  - finally this sets up the environment and calls the entrypoint of the
    interpreter
    - i.e., executes wine again with some memory rages reserved
- `__wine_main` again
  - `init_paths` again
  - because `$WINELOADERNOEXEC` is set, no `loader_exec`
  - set `RLIMIT_NOFILE` and `RLIMIT_AS` to max values
  - `virtual_init` sets up the address space
  - `init_environment` sets up the environment
  - calls `start_main_thread`
- `start_main_thread`
  - `server_init_process` connects to or spawns `wineserver`
  - `load_ntdll` calls `open_builtin_file` to open `ntdll.dll`
    - `open_dll_file` makes a request from the server; for a newly started
      server, this fails with `STATUS_DLL_NOT_FOUND`
    - it then falls back to dlopen built-in `ntdll.dll.so`
    - `load_ntdll_functions` sets `p__wine_set_unix_funcs` to
      `__wine_set_unix_funcs`
  - `load_libwine` dlopens `libwine.so.1.0`
  - `p__wine_set_unix_funcs` is `__wine_set_unix_funcs` and calls
    `process_init`
- `process_init`
  - `init_user_process_params`
    - `run_wineboot`
      - `RtlCreateUserProcess` -> `NtCreateUserProcess` -> `spawn_process`
      - the forked child calls `exec_wineloader` with
        - `argv[] = { "wine-preloader", "wine", "wineboot.exe" }`
  - `load_dll` to load `kernel32.dll`
  - `load_dll` again to load `foo.exe`

## Direct3D

- Built-in DLLs
  - `dxgi.dll` is needed by Direct3D 10, 11, and 12
  - wined3d provides `wined3d.dll` and is used by
    - `ddraw.dll` for Direct3D 7 and below
    - `d3d8.dll` for Direct3D 8
    - `d3d9.dll` for Direct3D 9
    - `d3d10.dll`, `d3d10core.dll`, and `d3d10_1.dll` for Direct3D 10
    - `d3d11.dll` for Direct3D 11
    - internally,
      - `wined3d.dll` calls to `wined3d.so` to cross win/lin bounary
      - `wined3d.so` translates d3d to gl or vk
  - vkd3d provides `libvkd3d.so` and is used by
    - `d3d12.dll` for Direct3D 12
    - internally,
      - `d3d12.dll` calls to `d3d12.so` to cross win/lin bounary
      - `d3d12.so` calls to `libvkd3d.so` to translate d3d to gl or vk
- (External) Native DLLs
  - dxvk provides native `d3d9.dll`, `d3d10.dll`, `d3d10core.dll`,
    `d3d10_1.dll`, `d3d11.dll`, and `dxgi.dll`
  - vkd3d-proton provides native `d3d12.dll`
  - they translate d3d to vk and call to `vulkan-1.dll`

## MiceWine

- <https://github.com/KreitinnSoftware/MiceWine-Application>
  - `MainActivity` starts `WelcomeActivity` to setup
    - `RootFSDownloaderFragment` retrieves rootfs list from
      <https://api.github.com/repos/KreitinnSoftware/MiceWine-RootFS-Generator/releases>
    - `RootFSDownloaderService` downloads the selected rootfs to `/data/data/com.micewine.emu/files/rootfs.rat`
    - `RatPackageManager` unpacks the rootfs to `/data/data/com.micewine.emu`
      - the rootfs is under `files/`
      - there are rat packages under `adrenoTools/`, `box64/`, `vulkanDrivers/`, and `wine/`
      - everything is built for android using ndk
    - `RatPackageManager` unpacks more rat packages to
      `/data/data/com.micewine.emu/packages/<Name>-<UUID>`
    - `runXServer` starts xserver
      - `CmdEntryPoint` loads `libXlorie.so` from the apk
      - `Java_com_micewine_emu_CmdEntryPoint_start` spawns a thread to run
        xserver
    - `createWinePrefix` creates prefix under
      `/data/data/com.micewine.emu/files/winePrefixes/default/drive_c`
      - `ShellLoader` loads `libmicewine.so` from the apk
      - `wineboot` invokes `env DISPLAY=:0 LD_LIBRARY_PATH=<...> PATH=<...> <more> WINEPREFIX=<...> box64 wine wineboot`
      - `Java_com_micewine_emu_core_ShellLoader_runCommand` invokes the shell cmd
- <https://github.com/KreitinnSoftware/MiceWine-RootFS-Generator>
  - it builds various projects using ndk
  - `mesa-zink` is a `OpenGLDriver` and provides `libGLX_mesa.so`
  - `mesa-vulkan-wrapper` is a `VulkanDriver` and provides `libvulkan_wrapper.so`
    - it is a vulkan driver that is a wrapper for `/system/lib64/libvulkan.so`
  - `mesa-vulkan-freedreno` is a `VulkanDriver` and provides `libvulkan_freedreno.so`
- IOW,
  - the rootfs
    - is compiled using NDK, not gcc
    - is not chroot'ed into
    - consists of wine and its deps
    - wine deps include mesa, vulkan loader, libglvnd, x11 libs, etc.
    - wine prefix additionally includes dxvk, vkd3d-proton, etc.
  - the xserver runs separately
    - mesa in rootfs is modified such that x11 wsi uses ahbs
      - xpixmaps are created from ahbs
    - xserver works with ahbs
  - take VVL for example
    - when `mesa-vulkan-freedreno` is used, we want to add VVL to rootfs and
      let khronos vulkan loader loads both
    - when `mesa-vulkan-wrapper` is used, we prefer the same
      - but since the wrapper loads the system vulkan, it is also possible to
        add VVL to android system and let android vulkan loader loads VVL

## WoW64

- wine is purely 64-bit on the linux side
  - all executables and all `.so` are 64-bit
  - unlikely before, where it ran in 32-bit mode for 32-bit apps
- a 32-bit win app runs in 32-bit cpu mode
  - a 32-bit syscall is handled by 32-bit `kernel32.dll`
    - which is lowered to 32-bit `ntdll.dll`
  - a 32-bit gui call is handled by 32-bit `user32.dll`
    - which is lowered to 32-bit `win32u.dll`
  - a 32-bit subsys call is handled by 32-bit `<subsys>.dll`, such as `d3d12.dll`
  - `32-bit ntdll.dll -> 32/64-bit wow64cpu.dll -> 64-bit wow64.dll -> 64-bit ntdll.dll -> 64-bit ntdll.so`
    - `wow64cpu.dll` switches the cpu to 64-bit
    - `wow64.dll` widens syscall params to 64-bit
  - `32-bit win32u.dll -> 32/64-bit wow64cpu.dll -> 64-bit wow64.dll -> 64-bit wow64win.dll -> 64-bit win32u.dll -> 64-bit win32u.so`
    - `wow64win.dll` widens gui params to 64-bit
  - `32-bit <subsys>.dll -> 32/64-bit wow64cpu.dll -> 64-bit <subsys>.so`
    - `<subsys>.so` is responsible for widening all params to 64-bit

## WoA

- similar to WoW64, there is `wowarmhw.dll` that switches the cpu to 64-bit mode like `wow64cpu.dll`
- there are additionally x86 cpu simulators
  - `xtajit.dll` simulates 32-bit x86 cpu
    - it replaces `wow64cpu.dll` in the call chain
    - it generates 64-bit arm instrs from 32-bit x86 instructions
    - when it hits a syscall, it pauses simulation and calls the 64-bit arm
      `wow64.dll -> ntdll.dll`
  - `xtajit64.dll` simulates 64-bit x86 cpu
    - it is similar to `xtajit.dll`
    - without arm64ec, the concept is the same
    - with arm64ec, it pauses simulation upon subsys call as well
- ARM64EC
  - a `<subsys>.dll` can be compiled for ARM64EC
    - it can only be loaded by X64 executable
    - the generated instrs are for ARM64, but with X64 ABI
    - there are also X64-fast-forward stubs
  - when the x64 process makes a x64 subsys call, it calls to the fast-forward stub
    - `xtajit64.dll` pauses cpu simulation, maps x64 regs to arm64 regs,  and
      calls the real function in `<subsys>.dll`
- ARM64X
  - it is fat and contains both ARM64EC and ARM64 code
