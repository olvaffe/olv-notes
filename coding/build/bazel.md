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
  - `--target_environment`
- <https://bazel.build/run/bazelrc>
  - `bazelrc` has lines such as `build --foo` and `build --bar`
  - `bazel run //main:hello-world` becomes `bazel run --foo --bar //main:hello-world`
    automatically
  - it can also has `build:baz --dah`, which is used when `--config=baz` is
    specified
- <https://bazel.build/docs/configurable-attributes>
  - `BUILD` has `select({"cond1": val1, "cond2": val2})` 
  - it also defines the conditions
    - `config_setting(name = "cond1", values = ...)`
    - this means, `cond1` is selected when cmdline matches `values`
- <https://bazel.build/concepts/platforms>
  - `bazel build --platforms=//:myplatform`
  - `platform(name = "myplatform", constraint_values = ["@platforms//os:linux"])`
    - `@platforms` refers to the canonical platforms
  - <https://github.com/bazelbuild/platforms>
    - `cpu`: `aarch64`, `x86_64`, etc.
    - `os`: `android`, `linux`, `windows`, `emscripten`, `chromiumos`, etc.
  - the legacy way is `bazel build --cpu=... --crosstool_top=...  --compiler=...`
