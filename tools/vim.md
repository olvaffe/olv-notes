Vim
===

## Files

- :help file-searching
  - downward search
    - `*` matches 0 or more characters
    - `**` matches directories
  - upward search
    - `;`

## ctags

- `:help tagsrch.txt`
  - `CTRL-]` and `CTRL-T` to jump to and back
  - `g]` to list all matches
  - `:tag` and `:tselect` accept regex
  - `set tags=./tags;`
    - `;` stands for upward search
- `tags` format
  - `{tagname}{TAB}{tagfile}{TAB}{tagaddress}{term}{fields}`
  - `tagname` is an identifier, such as the function name
  - `tagfile` is the file where `tagname` is defined
  - `tagaddress` is the vim excmd to jump to `tagname` in `tagfile`, such as
    `/^#define FOO(/`
  - `term` is `;"`, which starts a comment in vi, for backward compat
  - `fields` is a list of optional fields of the form
    `<Tab>{fieldname}:{value}`
    - `file:` means a static tag, such as a static function
    - `kind:{value}`, where `kind:` is optional, specifies the tag kind
      - see `ctags --list-kinds`
