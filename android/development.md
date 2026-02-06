Android Development
===================

## `scripts/lldbclient.py`

- usage
  - `-p <pid>` attaches to pid
  - `-n <name>` attaches to named process
  - `-r <cmd>` invokes the cmd and attaches to it
- `gdbrunner.start_gdbserver` starts gdbserver on dut
  - `/data/local/tmp/arm64-lldb-server gdbserver unix:///data/local/tmp/debug_socket ...`
  - it also `adb forward tcp:5039 localfilesystem:/data/local/tmp/debug_socket`
- `gdbrunner.start_gdb` starts lldb on host
  - `prebuilts/clang/host/linux-x86/clang-rFOO/bin/lldb.sh --source /tmp/BAR`
  - `settings append target.exec-search-paths $ANDROID_PRODUCT_OUT/symbols/<search-paths>`
    - `system/lib64/{,hw,egl,...}`
    - `vendor/lib64/{,hw,egl,...}`
    - `apex/com.android.runtime/bin` (real path of linker)
  - `target create $ANDROID_PRODUCT_OUT/symbols/<executable>`
  - `settings append target.source-map '' '$ANDROID_BUILD_TOP'`
  - `target modules search-paths add / $ANDROID_PRODUCT_OUT/symbols`
    - this is the search paths of dynamic libraries
    - while `target.exec-search-paths` is the search paths of executable
  - `gdb-remote 5039`

## `tools/winscope`

- <https://source.android.com/docs/core/graphics/winscope/overview>
- trace targets
  - `Event Log (CUJs)` enables perfetto `linux.ftrace` for system
    services/apps
  - `View Capture` enables perfetto `android.viewcapture`
