WINE
====

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

## Proton Environment Variables

- from `user_settings.sample.py`,
  - proton
    - `PROTON_USE_WINED3D=1` to use GL-based wined3d rather than dxvk for
      D3D{9,10,11}
    - `PROTON_NO_D3D11=1` to disable D3D11 entirely
    - `PROTON_NO_ESYNC=1` to disable eventfd-based in-process synchronization
    - `PROTON_NO_FSYNC=1` to disable futex-based in-process synchronization
  - wine
    - `WINEDEBUG=+timestamp,+pid,+tid,+seh,+debugstr,+loaddll,+mscoree`
  - dxvk
    - `DXVK_LOG_LEVEL=info`
    - `DXVK_HUD=devinfo,fps` for HUD
  - vkd3d
    - `VKD3D_DEBUG=warn`
  - wine-mono
    - `MONO_LOG_LEVEL=info`
    - `WINE_MONO_TRACE=E:System.NotImplementedException`
  - gstreamer
    - `GST_DEBUG=4`
