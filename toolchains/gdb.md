GDB
===

## `gdbserver`

* `set solib-absolute-prefix <root-path>"`
* `set solib-search-path <relative-paths>`
* `target remote :1234"`

## Threading

* `gdb` and `gdbserver` loads `libthread_db.so` at runtime to enable
  debugging of multithreaded applications
  * On Android,
