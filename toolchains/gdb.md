GDB
===

## `gdbserver`

* to run a program under `gdbserver`, `gdbserver :1234 prog args`
* to attach, `gdbserver --attach :1234 pid`
  * use `ps -AL` to find the tid, for a multi-threaded program

## cross/remote debugging

* `gdb -ex "target remote :1234" prog`
* `set sysroot <sysroot>`
* `set solib-absolute-prefix <sysroot>`
* `set debug-file-directory <debug-file-dir>`
* `set solib-search-path <relative-paths>`
* `set directories <source-file-dirs>`

## Threading

* `gdb` and `gdbserver` loads `libthread_db.so` at runtime to enable
  debugging of multithreaded applications
