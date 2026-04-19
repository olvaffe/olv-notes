lldb
====

## `platform` command

- `platform list` lists supported platforms
  - `host` talks to local platform
  - `remote-linux` talks to remote linux platform
  - `remote-android` talks to remote android platform
    - it sets up adb forwards automatically
- local debugging uses `host` platform
  - lldb manages host platform directly and there is no `lldb-server platform`
  - `run` spawns and talks to a `lldb-server gdbserver`
- remote debugging with `remote-linux` platform
  - the remote should have started lldb-server
    - `lldb-server platform --server --listen '*:1234'`
  - lldb manages remote platform via remote `lldb-server platform` server
    - `platform select remote-linux` selects `remote-linux` platform
    - `platform connect connect://<remote>:1234` connects to the remote
  - `run` uploads the executable, and spawns and talks to a `lldb-server gdbserver`

## `target` command

- `target create <executable>` creates a target
- `target modules`
  - `list` lists current modules (executable and shared libraries)
  - `search-path add` is the module maps
    - e.g., it maps `/` to symbol sysroot

## `process` command

- `process launch` launches a new process for the target
  - this is the same as `run`
- `process attach --pid pid` attaches to a running process
- `process connect --plugin gdb-remote connect://<remote>:<port>` connects to
  `lldb-server gdbserver`
  - this is the same as `gdb-remote` or `gdb`
  - it selects `remote-linux` platform without connecting to `lldb-server platform`

## `settings` command

- `settings list [<var>]` shows descriptions of matching variables
- `settings show [<var>]` shows current values of matching variables
- `settings append <var> <vals>` appends space-separated vals to variable
- `target` variables
  - `target.exec-search-paths` is the search paths for executable
  - `target.source-map` is the source file maps
    - e.g., it maps "" to source root
