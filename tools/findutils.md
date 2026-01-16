findutils
=========

## Overview

- <https://git.savannah.gnu.org/cgit/findutils.git>
  - `find`, `xargs`
  - `locate`, `updatedb`
- alternative `locate` and `updatedb` impls
  - <https://pagure.io/mlocate>
  - <https://plocate.sesse.net/>

## Expressions

- an expression composes of a sequence of operations
- an expression is evaluated from left to right against each file until the
  outcome is known
- global options always evaluate to true
  - these should be specified first
  - `-maxdepth` specifies max descend from the starting point
  - `-mount` stays on the current filesystem
- positional options always evaluate to true
  - they only affect tests that occur later
  - rarely used
- operators: `(`, `)`, `!`, `-a`, `-o`
  - `expr1 expr2` is the same as `expr1 -a expr2`
  - in `expr1 -a expr2`, expr2 is not evaluated if expr1 is false
- tests evaluate to true or false
  - `-amin` and `-atime` test atime
  - `-cmin` and `-ctime` test ctime
  - `-mmin` and `-mtime` test mtime
  - `-name` test basename
  - `-path` test fullname
- actions evaluate  to true or false with side-effect
  - `-exec` executes a cmd; true if cmd returns 0
  - `-print` prints fullname and newline; always true
    - this is implied unless the expr already has actions (that are not
      `-prune`/`-quit`)
  - `-prune` to avoid descend; always true
  - `-quit` quits immediately
