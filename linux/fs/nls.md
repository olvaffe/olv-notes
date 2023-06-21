Kernel NLS
==========

## API

- the header is `linux/nls.h`
- utf8 operations
  - these operations do not require a `nls_table`
    - some subsystems require NLS but do not require any table
  - `utf8_to_utf32`/`utf32_to_utf8` for to/from unicode
  - `utf8s_to_utf16s`/`utf16s_to_utf8s` for to/from UTF-16
- charset operations
  - a `nls_table` is required
  - `load_nls` loads the named table
    - `load_nls_default` returns the default table
  - `nls->char2uni`/`nls->uni2char` for to/from charset
  - `unload_nls` unlaods the table
- caseness operations
  - a `nls_table` is required
  - `nls_tolower`/`nls_toupper` for lower/upper
  - `nls_strnicmp` for case-insensitive comparison

## Internals

- `load_nls_default` calls `load_nls(CONFIG_NLS_DEFAULT)`
  - it falls back to `load_nls("default")` if misconfigured
- `CONFIG_NLS_CODEPAGE_437`
  - the module calls `register_nls(&table)` to register the cp437 table
