Bison
=====

## Parser

- The overall format of a `.y` input file is
  - `%{`
  - prologue
  - `%}`
  - declarations
  - `%%`
  - grammar rules
  - `%%`
  - epilogue
- `%pure-parser` makes the parser reentrant
- `%verbose-errors` produces more verbose error messages
- `%locations` enable location tracking
- `%initial-action` executed first every time `yyparse` is called
- `parse-param` and `lex-param` add a parameter to `yyparse` and `yylex`
- `%expect 0` expect zero grammar conflicts
- `%union {}` specifies all possible data types of semantic values
- `%token name` or `%token <type> name` declares a token type name (terminal symbol)
  - when the semantic value is used, `<type>` should be given
- `%type <type> nonterminal` declares a non-terminal symbol
- `%left op1` declares a toekn type name with precedence
  - in `expr op1 expr op1 expr`, it is grouped as `(expr op1 expr) op1 expr`
  - if `op2` is declared after `op1`, then `expr op1 expr op2 expr` is grouped
    as `expr op1 (expr op2 expr)`
  - `-` may be used as an uniary op (high prec) or binary op (low prec).  The
    precedence depends on the context, and that can be achieved by `%prec`
- Special Features for Use in Actions
  - `$$` semantic value for the grouping made by current rule
  - `$n` semantic value for the n-th component
  - `YYABORT` return immediately from `yyparse` with error
  - `YYACCEPT` return immediately from `yyparse` with success
  - `yychar` the lookahead token
  - `yylloc` the lookahead token location
  - `yylval` the lookahead token value
  - `@$` textual location of the grouping made by current rule
  - `@n` textual location of the n-th component
- it calls `yylex` to scan the input.  The function must be written, or
  generated from flex.
- the parsing algorithm are shift/reduce automata.
  - as tokens are read, they are pused onto the parser stack along with the
    semantic values.  this operation is traditionally called shifting.
  - when the last N tokens match a grammar rule, they can be combined.  This is
    called reduction.  Running the rule's action is part of the process of
    reduction.
  - lookahead token: when the last N tokens match a rule, they are not always
    reduced immediately.  When a token is read, it becomes the lookahead token.
    Parser will decide how many reductions on the stack to perform before
    shifting the lookahead token.  The lookahead token is stored in `yychar`.
    Its semantic value and location are stored in `yylval` and `yylloc`.
  - When shift and reduce are both valid, it is called a shift/reduce conflict.
    Bison prefers shift, unless otherwise directed by operator precedence.
  - A reduce/reduce conflict occurs when there are more than one way to reduce
    the tokens.
- start symbol

## Flex and `yylex`

- The overall format of a `.l` input file is
  - definitions
  - `%%`
  - rules
  - `%%`
  - user code
- definitions
  - e.g. `DIGIT [0-9]`.  The defition may be referred to later using `{DIGIT}`.
  - indented text or text enclosed in `%{...%}` will be copied to the output
    verbatim
- rules
  - `pattern action`
  - pattern use `"escaped"` to escape
- Options
  - `never-interactive` do not read a char at a time
  - `noyywrap` stop on EOF
  - `lineno`
  - `prefix=XXX` use XXX instead of yy as the prefix of global variables
- Once a match is determined, the token is made available in `yytext` and its
  length in `yyleng`.  action is then executed.
- Actions
  - `input()` or `yyinput()` reads the next char in the input stream
    - `#define YY_NO_INPUT` to kill the function if not used
  - `ECHO` copies yytext to the scanner's output. 
  - `BEGIN` followed by the name of a start condition places the scanner in
    the corresponding start condition (see below). 
  - `REJECT`
- misc macros
  - `YY_USER_INIT` is executed once before any scan action
  - `YY_USER_ACTION` is executed just before every scan action
- Reentrant
  - enabled by `%option reentrant`
  - all functions will take one additional argument, `yyscan_t yyscanner`
  - all global variables become macros
    - `#define yytext yyscanner->yytext_r`
  - `yylex_init` and `yylex_destroy` must be called before/after `yylex`
  - user data may be stored in `yyextra`.  Its value is determined at
    `yylex_init_extra`
    - the type is determined by `%option extra-type="XXX"`
- Start Conditions
  - enabled by `%option stack`
  - a condtion is declared by `%s cond1` or `%x cond2`
    - the former is inclusive and the latter is exclusion
  - the current condition is specified by `BEGIN` or
    `yy_push_state/yy_pop_state`
- Multiple Input Buffers
  - to support something like `#include <a.h>`
  - a in-memory C-string can also be wrapped in a buffer
    - `yy_scan_string`
- Bison Bridge
  - `bison-bridge` generate C code for use with bison.  It adds additional
    `YYSTYPE yylval`
  - `bison-locations` generate C code for use with bison `%locations`.  it adds
    additional `YYLTYPE yylloc`
