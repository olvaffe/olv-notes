Mesa GLSL2
==========

## Sources

* `main.cpp` is used by the standalone compiler
* `opt_*.cpp`, `lower_*.cpp`, and `loop_*.cpp` are for lowerings/optimizations
* `linker.cpp` is for shader linking
* `builtin_function.cpp` defines shaders of built-functions.  They will be
  linked to app shaders after linking.
* `ir_reader.cpp` and `s_expression.c` are used to read built-in functions
* `ir*.cpp` and `builtin_variables.h` are for IR
* `ast_*.cpp` and `hir_field_selection.cpp` are for AST and its methods for HIR
  translations
* `glsl_symbol_table.cpp`, `glsl_types.cpp` and `builtin_types.h` are for the
  symbol table and data types
* `glsl_lexer.ll`, `glsl_parser.yy`, and `glsl_parser_extras.cpp` are for the
  parser
* To study the sources, start from the parser and go up through this list

## GLSL2

* Lex and Yacc
  * the lexer will generate
    * `_mesa_glsl_lex_init_extra` to create a reentrant scanner with user data
    * `_mesa_glsl__scan_string` to specify the input
    * `_mesa_glsl_lex` to lex
  * the parser will generate `_mesa_glsl_parse`
    * it calls `_mesa_glsl_lex` for input
* The lexer is simple
* The parser
  * the results are `ast_node`s stored in `state->translation_unit`
    * An `ast_node` at this level is either from `declaration` or
      `function_definition`
  * Examples of `declaration`s are
    * `float dot2(vec2, vec2);`
    * `precision highp float;`
    * `uniform float a = 0.3, b;`
    * `struct pair { int a; int b; };`
    * the last two are also known as `init_declarator_list`
      * the type of the semantic value is `ast_declarator_list`
  * Example of `function_definition` is
    * `void main(void) { ... }`
* Memory Management
  * talloc
  * `_mesa_new_shader_program` creates a `gl_shader_program` that is also a
    talloc context
  * `_mesa_new_shader` creates a `gl_shader` that is also a talloc context
* `glCompileShader` calls `_mesa_glsl_compile_shader`
  * A temp `_mesa_glsl_parse_state` is created for compilation
  * `preprocess` passes the shader source through the preprocessor
  * the lexical scanner is created for the source
  * `_mesa_glsl_parse` is called to construct the AST
    * `_mesa_glsl_initialize_types` is called to add data types to the symbol
      table
  * `_mesa_ast_to_hir` is called to construct the HIR
    * `_mesa_glsl_initialize_variables` is called to add built-in variables to
      the `exec_list` and the symbol table
    * `_mesa_glsl_initialize_functions` is called to add built-in functions
      * built-in functions are lowered to use only `ir_expression_operation`
  * the HIR is validated and optimized
  * finally, `ctx->Driver.CompileShader = _mesa_ir_compile_shader` is called. It
    is no-op.
* `glLinkProgram` calls `_mesa_glsl_link_shader`
  * `link_shaders` is called for linking
    * all vertex shaders are linked together by `link_intrastage_shaders`
    * same to all fragment shaders
  * `ctx->Driver.LinkShader = _mesa_ir_link_shader` is called.  The IRs are
    lowered and a `gl_program` is created from each `gl_shader`


## GLSL2 AST

* class hierarchy

    ast_node -> ast_expression -> ast_expression_bin
                               \> ast_function_expression
             -> ast_type_specifier
             -> ast_fully_specified_type
             -> ast_declaration
             -> ast_declarator_list
             -> ast_struct_specifier
             -> ast_parameter_declarator
             -> ast_function
             -> ast_function_definition
             -> ast_jump_statement
             -> ast_selection_statement
             -> ast_iteration_statement
             -> ast_expression_statement
             -> ast_compound_statement

* In the top-level, there are
  * `ast_function` (e.g., `float pow2(float, float);`)
  * `ast_type_specifier` (e.g., `precision lowp float;`)
  * `ast_declarator_list` (e.g., `uniform int a, b[3];`)
  * `ast_function_definition`
* `int var[3], b = 3;` is translated to an `ast_declarator_list`, which consists
  of
  * an `ast_fully_specified_type`
  * a list of `ast_declaration`
    * each declaration consits of an identifier, two expressions for array
      size and initializer
* `struct s { int a, b; float c; }` is translated to an `ast_struct_specifier`,
  which consists of
  * an identifier
  * a list of `ast_declarator_list`
* `int main(void) { ... }` is translated to an `ast_function_definition`, which
  consists of
  * an `ast_function` for the prototype, which consists of
    * an `ast_fully_specified_type`
    * an identifier
    * a list of `ast_parameter_declarator`
  * an `ast_compound_statement` for the body
* `continue; break; return val; discard;` are translated to `ast_jump_statement`
* IF is translated to `ast_selection_statement`, which consists of
  * an `ast_expression` for the condition
  * two statesments (may be compound) for then and else
* DO-loop, WHILE-loop, and for-loop are translated to `ast_iteration_statement`
* `expression;` is translated to `ast_expression_statement`, which consits of an
  expression
* `pow(2.0, 3.0)` is translated to `ast_function_expression`, which is an
  `ast_expression`
  * `oper` is `ast_function_call`
  * `subexpressions[0]` is `pow`
  * `expressions` is the arguments

## GLSL2 IR

* class hierarchy

    exec_node -> ir_instruction -> ir_rvalue -> ir_expression
                                             -> ir_call
                                             -> ir_texture
                                             -> ir_swizzle
                                             -> ir_dereference -> ir_dereference_variable
                                                               -> ir_dereference_array
                                                               -> ir_dereference_record
                                             -> ir_constant
                                -> ir_variable
                                -> ir_function_signature
                                -> ir_function
                                -> ir_if
                                -> ir_loop
                                -> ir_assignment
                                -> ir_jump -> ir_return
                                           -> ir_loop_jump
                                           -> ir_discard
* A function is translated to an `ir_function`, which consists of
  * an identifier
  * multiple `ir_function_signature`s, which consists of
    * a `glsl_type` for the return type
    * zero or more `ir_variable`
    * body

## GLSL2 HIR

* AST is translated to HIR by `_mesa_ast_to_hir`
* Examples of top-level `ast_node`s' hir()
  * `ast_type_specifier` for precision keyword or struct declaration
    * `ast_type_specifier::hir` first checks if it is a precision statement or
      struct declaration
    * precision statement is ignored
    * struct declaration is translated by `ast_struct_specifier::hir` which adds
      a new type to the symbol table
  * `ast_declarator_list` for variable declaration
    * for each variable, an `ir_variable` is added to the symbol table.  If
      there is an initializer for the variable, the initializer is added to the
      instructions
  * `ast_function` for function declaration
    * it adds an `ir_function` to the instructions as well as the symbol table
  * `ast_function_definition` for function definition
    * it translates the prototype as above first
    * the scope of the symbol table is incremented and the parameters are added
      as `ir_variable`s
    * the body is then translated
* symbol table
  * after `#version <ver>` is parsed, `_mesa_glsl_initialize_types` is called
    * `glsl_type` may represent builtin types, structures, arrays, or functions
  * before translating to HIR, `_mesa_glsl_initialize_variables` is called
    * it adds `ir_variable`s to the symbol table as well as the instruction
      list
    * a variable may be an in(/out), uniform, or constant
  * so is `_mesa_glsl_initialize_functions`
    * it adds `ir_function`s to the symbol table as well as the instruction list
    * built-ins are written in GLSL IR
    * they are parsed at runtime using `ir_reader`
      * `(function IDENTIFIER (signature RET_TYPE (parameters PARRMS) BODY))`
      * there may be multiple signatures (overloading) for a function
  * symbol table is a hash table for types, variables, and functions
  * it has scopes; symbols in an outer space can be accessed; symbols in an
    inner space cannot be accessed
* `_mesa_glsl_initialize_types`, `_mesa_glsl_initialize_variables` and
  `_mesa_glsl_initialize_functions` update the symbol table.  The latter two
  also inserts variable declarations and function prototypes to the instruction
  list.
  * as for function bodies, they are added to `builtins_to_link`
* `_mesa_ast_to_hir` calls `hir` method of each AST node
  * `ast_expression` to `ir_rvalue`
    * map `ast_xxx` to `ir_xxxop_xxx`

## GLSL2 Linking

* shaders are linked by `link_shaders`
* `move_non_declarations` moves initializers to `main`
  * e.g., initializer for a global variable `int global_var = 3 * 5;`

