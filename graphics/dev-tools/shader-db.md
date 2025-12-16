Mesa shader-db
==============

## GL

- `MESA_SHADER_CAPTURE_PATH=<path>` dumps all program sources to `<path>` in
  `.shader_test` format
  - `link_program` handles the dumping
- `./run <path>` compiles all `.shader_test` files in `<path>` and its
  subdirs, and prints the stats
  - it sets various mesa-specific envs
    - `allow_glsl_extension_directive_midshader=true`
    - `allow_glsl_builtin_variable_redeclaration=true`
    - `allow_glsl_120_subset_in_110=true`
    - `allow_glsl_compat_shaders=true`
    - `shader_precompile=true` causes drivers to precompile shaders
      - that is, pre-generate shader variants using guessed keys
      - this is usually the default behavior for drivers as it avoids stutters
        in real games
    - `MESA_SHADER_CACHE_DISABLE=true` disables disk cache
      - this either forces compiles or avoids caching the binaries
    - `GALLIUM_THREAD=0` disables TC
      - because TC disables synchronous debug messages in
        `tc_set_debug_callback`
    - drivers-specific envvars to make sure stats are printed
      - this requires debug builds for some drivers
  - it enables debug messages, which are how some drivers report stats
    - `glEnable(GL_DEBUG_OUTPUT);`
    - `glEnable(GL_DEBUG_OUTPUT_SYNCHRONOUS);`
- `./report.py <old-stats> <new-stats>` to report deltas
  - use `nv-report.py` for nouveau; `si-report.py` for radeonsi
- iris stats
  - `iris_create_shader_state` calls `iris_schedule_compile` to schedule a
    compile
  - `iris_compile_shader` compiles the shader on another thread
    - the backend compiler calls `shader_debug_log`, which points to
      `iris_shader_debug_log`, to log stats as debug messages
  - `u_async_debug_drain` prints all debug messages
    - note that `tc_set_debug_callback` may disable debug messages and must be
      disabled with `GALLIUM_THREAD=0`

## VK

- `VK_LAYER_fossilize` layer dumps an `.foz` file
  - `fossilize.sh` is a convenient shell script
- `fossil_replay.sh` invokes `fossilize-replay --enable-pipeline-stats` to
  replay `.foz` files
  - this requires `VK_KHR_pipeline_executable_properties`
- `report-fossil.py <old-stats> <new-stats>` to report deltas
