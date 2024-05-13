Bazel
=====

## Getting Started

- <https://bazel.build/start/cpp>
  - `WORKSPACE`
  - `BUILD`
  - `bazel build //main:hello-world`
  - `bazel-bin/main/hello-world`
- `bazel build` options
  - `-c {fastbuild|opt|dbg}`
    - `fastbuild` is the default; no opt and minimal debug info
    - `opt` adds `-O2 -DNDEBUG`
    - `dbg` adds `-g`
  - `--config=<config>` selects a config from the rc file
    - `/etc/bazel.bazelrc`
    - `.bazelrc` and `tools/bazel.rc` next to `WORKSPACE`
    - `$HOME/.bazelrc`
  - `--copt=<copt>` passes a compiler option to the compiler
- <https://bazel.build/docs/configurable-attributes>
  - `BUILD` has `select({"cond1": val1, "cond2": val2})` 
  - it also defines the conditions
    - `config_setting(name = "cond1", values = ...)`
    - this means, `cond1` is selected when cmdline matches `values`
