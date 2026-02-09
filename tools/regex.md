Regular Expression
==================

## RE 101

- literals
  - most characters are literals
  - special characters can be escaped to become literals
    - e.g., `.`, `*`, `[`, `]`, `\`', etc. can be escaped
  - regular characters can be escaped to become alternative literals
    - `\r` is carriage return
    - `\t` is tab
- character classes
  - `[xyz]` is postive class
  - `[^xyz]` is negative class
  - `[x-z]` is range
  - `[[:name:]]` is posix named set
    - e.g., `alpha`, `digit`, `alnum`, etc.
  - `.` is any character
  - `\d` is `[[:digit:]]` (`\D` is negative)
  - `\s` is `[[:space:]]` (`\S` is negative)
  - `\w` is `[[:word:]]` (`\W` is negative)
- quantifiers
  - `?` is 0 or 1 (`??` is lazy)
  - `*` is 0 or more (`*?` is lazy)
  - `+` is 1 or more (`+?` is lazy)
  - `{n}` is n
  - `{n,m}` is n to m (`{n,m}?` is lazy)
- anchors
  - `\b` is word boundary (`\B` is negative)
  - `^` is start of line
  - `$` is end of line
- alternation
  - `expr1|expr2` is either expr1 or expr2
- grouping
  - `(...)` is a capture group
  - `(?<name>...)` is a named capture group
  - `(?:...)` is a non-capture group
- substitution
  - `$$` is literal dollar sign
  - `$n` is group n
  - `$name` is named group
  - `$0` is entire match
  - \$\` is pre-match
  - `$'` is post-match

## VIM and BRE

- special characters
  - `?`, `+`, `|`, `(`, and `)` are literals unless escaped
- anchors
  - `\<` and `\>` are similar to `\b`
  - if VIM, `\b` is backspace literal
- substitution
  - `\n` is group n
- options
  - if VIM, `\c` is case-insensitive
