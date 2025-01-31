AWK
===

## awk

- <https://git.savannah.gnu.org/cgit/gawk.git>
- awk scans over a text file
  - it treats each line as a record
  - it breaks each record into fields
  - awk 'END{print NR}' is the same as `wc -l`
    - `NR` stands for number of record
- `awk KEYWORD{ACTIONS}`
  - keywords
    - if `BEGIN`, `ACTIONS` are executed before the first record
    - if `END`, `ACTIONS` are executed after the last record
    - if `/regex/`, `ACTIONS` are executed after each record matched by
      `regex`
    - if omitted, `ACTIONS` are executed after each record
  - built-in variables
    - `$0` is the record
    - `$1` and so on are the fields
  - built-in functions
    - `print`
    - `printf`
