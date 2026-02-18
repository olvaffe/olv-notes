Wine
====

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
  - wined3d provides `wined3d.dll` and is used by
    - `ddraw.dll` for Direct3D 7 and below
    - `d3d8.dll` for Direct3D 8
    - `d3d9.dll` for Direct3D 9
    - `d3d10.dll`, `d3d10core.dll`, and `d3d10_1.dll` for Direct3D 10
    - `d3d11.dll` for Direct3D 11
  - vkd3d provides `libvkd3d.so` and is used by
    - `d3d12.dll.so` for Direct3D 12
  - `dxgi.dll` is needed by Direct3D 10, 11, and 12
- (External) Native DLLs
  - dxvk provides native `d3d9.dll`, `d3d10.dll`, `d3d10core.dll`,
    `d3d10_1.dll`, `d3d11.dll`, and `dxgi.dll`

## DXVK

- `DxvkSubmissionQueue`
  - `submit` or `present` adds a gpu job to `m_submitQueue`
  - there is a `dxvk-submit` thread
    - it retrieves gpu jobs from `m_submitQueue`, submit them to `VkQueue`,
      and moves them to `m_finishQueue`
  - there is also a `dxvk-queue` thread
    - it waits on the gpu jobs to complete, free the associated resources, and
      recycles the jobs for reuse

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
