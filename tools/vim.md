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
- `ctags`
  - ctags deterimines the language of a file and invokes the corresponding
    parser
    - `--list-maps` shows how file names are mapped to languages
    - `--map-<LANG>` changes the mappings
      - `--map-C='+.foo'` maps `*.foo` as C
      - `--map-C='+(macros.*)'` maps `macros.*` to C (rather than RpmMacros)
    - `--languages` changes languages to support
      - `--languages=-RpmMacros` disables RpmMacros
  - `-I FOO+`
    - when an identifier is defined as `void FOO(bar) func_name`, this tells
      the parser toskips `FOO(bar)`

## Language Server Protocol

- Language Server Protocol, LSP, is a JSON-RPC-based protocol
  - base protocol
    - `Content-Length: ...\r\n`
    - `\r\n`
    - `{`
    - `   "jsonrpc": "2.0",`
    - `   "id": 1,`
    - `   "method": "textDocument/completion",`
    - `   "params": {`
    - `     ...`
    - `   }`
    - `}`
  - Document Synchronization
    - `textDocument/didOpen` sends the contents of a newly opened file to the
      server
    - `textDocument/didChange` sends changes to a file to the server
    - `textDocument/didClose` notifies the server about file close
  - Language Features
    - `textDocument/declaration` resolves a symbol to the location of
      declaration
    - `textDocument/definition` resolves a symbol to the location of
      definition
    - `textDocument/rangeFormatting` formats a range of the document
  - Workspace Features
    - `workspace/symbol` lists all symbols matching the string
  - Window Features
    - `window/showMessage` sends a message from the server for display
    - `window/showDocument` sends a uri from the server for display
- Common Features
  - Errors and warnings
    - show compile errors as you type
    - suggest fixes for compile errors
    - linting (e.g., `clang-tidy`)
  - Code completion
    - suggestions to compele the current variable/function/method
    - suggestions for includes and namespace prefices
    - show parameters expected by a function
  - Cross-references
    - Find definition/declaration of a symbol
    - Find references of a symbol
  - Navigation
    - the structure of a file (which namespaces/classes/structs/functions are
      nested in where) is parsed for quicker navigation and for jumping to
      symbols
  - Hover
    - hover over a symbol will show its type, doc, and definition
  - Formatting
    - format selection or file (e.g., `clang-format`)
  - Refactoring
    - rename a symbol renames all references to it
- vim requires a plugin such as `vim-lsp`
- neovim has native support for LSP
  - in a `FileType` autocmd, call `vim.lsp.start()` to start an lsp client and
    attaches the buffer to the client
  - it sets these options if they are not customized yet
    - `omnifunc` is set to `vim.lsp.omnifunc()` (`CTRL-X_CTRL-O`)
    - `tagfunc` is set to `vim.lsp.tagfunc()` (`CTRL-]`)
    - `formatexpr` is set to `vim.lsp.formatexpr()` (`gq`)
    - `K` is mapped to `vim.lsp.buf.hover()`
  - the recommended way is to use `lspconfig` plugin
    - `require('lspconfig').clangd.setup{}`
      - `cmd` is `clangd`
      - `filetypes` is `{ 'c', 'cpp', 'objc', 'objcpp', 'cuda', 'proto' }`
      - `root_dir` is a function to find the root dir of the current file
        - it find the directory containing `compile_commands.json` (or
          several other files)
          - `ln -sf out/compile_commands.json` at the root dir for this to
            work
        - it also finds the parent of `.git` directory
