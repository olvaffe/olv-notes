Android Linker
==============

## Linker

* `.text` and entry point is set to `LINKER_TEXT_BASE` 0xb0000100
* entry function `_start` calls `__linker_init`
* `__linker_init` returns program's entry point and `_start` bl to it.
* all symbols are prefixed with `__dl_`

## Workflow

* `solist` begins with `libdl_info`, which is a faked `FLAG_LINKED` `libdl.so`
* `__linker_init` calls `link_image`
* in `link_image`, for each `DT_NEEDED` library, `find_library` is called to
  `load_library` the library and `init_library` (which calls `link_image` and recurses).
  e.g. `/system/bin/ls` uses `liblog.so`, which uses `libc.so`, which uses `libdl.so`
* after `reloc_library` is called in `link_image`, the `si` is marked `FLAG_LINKED`.
