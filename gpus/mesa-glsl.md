GLSL
====

## Ref
* <http://www.lighthouse3d.com/opengl/glsl/index.php?ogluniform>

## GLSL and ARB_fragment_program

* Fixed function program generator generates GLSL IR, and calls
  `_mesa_glsl_link_shader`
* `glLinkProgram` also calls `_mesa_glsl_link_shader`
* `_mesa_glsl_link_shader` calls `drv->LinkShader`, which may
  * convert GLSL IR to mechine code, or
  * convert GLSL IR to MESA IR
* `glProgramStringARB` creates a MESA IR program.

## How to Create Shader?

* Code snippet

    char * my_fragment_shader_source;
    char * my_vertex_shader_source;

    // Get Vertex And Fragment Shader Sources
    my_fragment_shader_source = GetFragmentShaderSource();
    my_vertex_shader_source = GetVertexShaderSource();

    GLenum my_program;
    GLenum my_vertex_shader;
    GLenum my_fragment_shader;

    // Create Shader And Program Objects
    my_program = glCreateProgramObjectARB();
    my_vertex_shader = glCreateShaderObjectARB(GL_VERTEX_SHADER_ARB);
    my_fragment_shader = glCreateShaderObjectARB(GL_FRAGMENT_SHADER_ARB);

    // Load Shader Sources
    glShaderSourceARB(my_vertex_shader, 1, &my_vertex_shader_source, NULL);
    glShaderSourceARB(my_fragment_shader, 1, &my_fragment_shader_source, NULL);

    // Compile The Shaders
    glCompileShaderARB(my_vertex_shader);
    glCompileShaderARB(my_fragment_shader);

    // Attach The Shader Objects To The Program Object
    glAttachObjectARB(my_program, my_vertex_shader);
    glAttachObjectARB(my_program, my_fragment_shader);

    // Link The Program Object
    glLinkProgramARB(my_program);

    // Use The Program Object Instead Of Fixed Function OpenGL
    glUseProgramObjectARB(my_program);
* `glCreateShader` creates a `struct gl_shader`
* `glCompileShader` invokes `_slang_compile`
  * A program has parameters, varyings, and attributes
  * 
* `glCreateProgram` creates a `struct gl_shader_program`
* `glLinkProgram` invokes `_slang_link`
* A shader object is a `struct gl_shader` and is created by
  `_mesa_create_shader`.
  * It has `Source` for local copy of the source code
* A program object is a `struct gl_shader_program` and is created by
  `_mesa_create_program`.
  * Attached shaders are stored in `Shaders`.
* Linking a program object
  * is done in `_slang_link`
  * calls `_mesa_new_uniform_list` for `Uniforms`
  * calls `_mesa_new_parameter_list` for `Varying`
  * calls `_mesa_ir_link_shader` to link shaders, resolve symbols, etc.
  * links uniforms and varying.
* Inputs and outputs
  * `InputsRead` denotes registers/varying that are read.  Possible flags in
    `gl_frag_attrib`.  Only for fragment shader?
  * `OutputsRead` denotes registers/varying that are written.  Possible flags in
    `gl_vert_result` or `gl_frag_result`, depending on the shader type.
  * For example, a vertex shader must has `VERT_RESULT_HPOS` set in its
    `OutputsRead`.
* Using a program updates `ctx->Shader.CurrentProgram` and remembers the states
  * state update is lazy and is done in `update_program`.
  * `ctx->FragmentProgram._Current` and `ctx->VertexProgram._Current` point to
    the new program's shaders.
  * `ctx->Driver.BindProgram` is called.

## `ARB_shader_objects`

* The idea of generic object is deprecated.  The "Object" in the function names
  are dropped in OpenGL 2.0.
* Generic Object
  * represented by `GLhandleARB`
  * some objects are containers to other objects.
    * `glAttachObjectARB` to add an object to a container
    * `glDetachObjectARB` to remove an object from a container
* Shader Object
  * created by `glCreateShaderObjectARB`
  * source code is specified by `glShaderSourceARB`
  * compiled by `glCompileShaderARB`
* Program Object
  * created by `glCreateProgramObjectARB`
  * linked by `glLinkProgramARB`
  * installed by `glUseProgramObjectARB`
* Uniform Variables
  * `glGetUniformLocationARB` to find the location of a uniform
  * ...
* In Mesa, a shader object is represented by `gl_shader` and created by
  `drv->CreateShader`.  A program object is represented by `gl_shader_program`
  and created by `drv->Createprogram`.
  * It is different from a program as defined by vertex/fragment program, which
    is represented by `gl_program` and created by `drv->NewProgram`.

## Qualifiers

* Suppose `uniform float specIntensity;` is defined in a shader.  It can be
  modified from C by
  * `GLint loc = glGetUniformLocation(prog,"specIntensity");`
  * `glUniform1f(loc,0.8f);`
* Suppose `attribute float height;` is defined in a shader.  It can be modified
  from C by
  * `GLint loc = glGetAttribLocation(prog,"height");`
  * `glVertexAttrib1f(loc, 2.0f);`
  * Alternatively, one may bind an attribute to some location through
    `glBindAttribLocation(prog, 0, "height");`.
* Variable Qualifiers
  * `const` gives compile time constant
  * `attribute` gives a variable that can be changed per-vertex, that are passed
    from C to vertex shaders.  It is read-only in vertex shaders.
  * `uniform` gives a variable that can be changed per-primitive (outside
    begin/end), that are passed from C to vertex and fragment shaders.  it is
    read-only in vertex and fragment shaders.
  * `varying` gives a variable that is written by vs and read by fs.

## Implementation of Qualifiers

* to modifiy an uniform, call `_mesa_uniform`
  * The uniform is looked up and the corresponding parameters in fs and vs are
    updated.
* attribute is stored per-vertex, just like any other attributes
  * It is also called generic attributes
  * There is a limitation on the number (usually 16) of generic attributes
    `glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &MaxVertexAttribs)`

## TGSI

* Every token is 32bits.  Some types are longer than 32bits and take several
  tokens.
  * E.g. A full declaration takes 3 tokens: declaration itself, range, and the
    semantic.
    * Every declaration consists of one or more registers.  The range gives the
      indices of the registers.  No two declarations of the same file can have
      conflicting range.
    * Input/Ouput declarations consists one a single register, representing an
      attribute (4 channels).
    * The semantic name of a declaration gives color/fog/generic/etc.  The
      semantic index distingushes primary/secondary color or which texture units
      or which varying.
    * For, say, temporary declaration, there is no semantics.
* When creating a vs or fs, a series of tokens are parsed and the results are
  stored in a  `struct tgsi_shader_info`.
  * See `draw_create_vs_exec`.
* When preparing a vs, `tgsi_exec_machine_bind_shader` is called.
  * Instructions, declarations, and immediates are splitted and stored in the
    machine.
* When running a vs, inputs, outputs and constants are supplied.
  * The inputs should fit the declarations.  Both the number and
    order should fit.
  * `tgsi_exec_machine_run` is called to start processing.  It works on 
    4 vertices at a time.
* The outputs of vs is post-processed and emitted to hw.  The pipe driver calls
  `draw_find_vs_output` to know the order of the outputs and builds a
  `struct vertex_info`.  The emission is determined by the vertex info.
* The pipe driver rasterizes the primitives and passes the fragments to fs in
  quads.
* The data passed to vs are input, output, and constant.  The data passed to fs
  are `struct quad_header`.
  * Compare `vs_exec_run_linear` and `shade_quad`.
  * The i/o requirements of vs and fs are determined by parsing the declarations.
  * A `struct vertex_info` is built to help translate vs outputs to fs inputs.
  * An fs input is an interpolation of a vs output.
    * It is stored in a `struct tgsi_interp_coef`.
    * `a0` is the value of the bottom-left fragment of the primitive
      rasterization.
    * The value of fragment `(x, y)` is `a0 + x * dadx + y * dady`.
  * See `sp_setup.c`.
* The vertex buffer is fetched for vs; the output of vs is emitted to fs
  * In this view, let's examine the formats.
  * As can be seen in `setup_non_interleaved_attribs`, the state tracker knows
    the format of vs inputs and vertex buffer to be fetched is made to be in
    that format.
  * As can be seen in `draw_pt_emit_prepare`, and
    `i915_vbuf_render_get_vertex_info` or `softpipe_get_vertex_info`, the
    emission knows the format of vs outputs and fs inputs.  It makes sure fs
    gets what it wants in non-vbuf mode.  In now default vbuf mode, the emission
    is pass-through.  That is, it emits what it receives.  fs inputs are
    determined in `setup_tri_coefficients`, `setup_line_coefficients` etc.

## i965c

* input is FP source, `_mesa_parse_arb_fragment_program` to MESA IR,
  `ProgramStringNotify` to notify the new MESA IR
* input is GLSL source, `_mesa_glsl_compile_shader` to GLSL IR,
  `ctx->Driver.LinkShader` to link the shaders and translate to MESA IR
* When `brw_validate_state` is called,
  * `brw_vs_emit` is called to translate vertex program/shader MESA IR to
    machine code
  * `brw_wm_fs_emit` or `brw_wm_non_glsl_emit` are called to translate fragment
    program/shader MESA/GLSL IR to machine code
    * the new backend is used for GLSL with GLSL IR
    * the old backend is used for FP with MESA IR
    * there is plan to translate ARB programs to GLSL IR


## Optimizations

* `glCompileShader` compiles the shader source and calls
  `do_common_optimization` indirectly to optimize an unlinked shader
  * it is done to reduce the size of IRs and save linking time later
* `glLinkProgram` calls `do_common_optimization` indirectly to optimize a linked
  program
  * it is done before assigning storages to attributes, uniforms, and varyings.
* For most drivers, `Driver.LinkShader` converts GLSL IR to MESA IR with
  lowerings and optimizations
  * it calls lowerings and `do_common_optimization` before conversion
  * it calls `_mesa_optimize_program` after conversion
* For i965c, `Driver.Linker` is set to `brw_link_shader`
  * it performs its own lowerings and optimizations for fragment shaders before
    performing the generic ones
* Before drawing, drivers validate its states and do codegen if necessary
* for i965c,
  * VS codegen is done by `do_vs_prog`
  * FS codegen is done by `do_wm_prog`
* for r300c,
  * VS codegen is done by `r300SelectAndTranslateVertexShader`
  * FS codegen is done by `r300SelectAndTranslateFragmentShader`


## C Terminologies

* `http://msdn.microsoft.com/zh-tw/library/ks1txk8c.aspx`
  * `declaration:`
    * `declaration-specifiers init-declarator-list_opt ;`
  * `declaration-specifiers:`
    * `storage-class-specifier declaration-specifiers_opt`
    * `type-specifier declaration-specifiers_opt`
    * `type-qualifier declaration-specifiers_opt`
  * `init-declarator-list:`
    * `init-declarator`
    * `init-declarator-list , init-declarator`
  * `init-declarator:`
    * `declarator`
    * `declarator = initializer`
  * For example, `static const int *fp;`
    * `declaration-specifiers`: `static const int`
      * `storage-class-specifier`: `static`
        * the other storage class specifiers are `extern`, `register`, `typedef`
      * `type-qualifier`: `const`
        * the other type qualifier is `volatile`
      * `type-specifier`: `int`
    * `init-declarator-list`: `*fp`
      * one declarator with no initializer in this example
      * the declarator is modified by `*` to declare a pointer
      * it may also be modified by `[]` or `()` to declare an array or a
        function
  * `declarator:`
    * `pointer_opt direct-declarator`
  * `direct-declarator:`
    * `identifier`
    * `( declarator )`
    * `direct-declarator [ const-expression_opt ]`
    * `direct-declarator ( parameter-list-type )`
  * `struct-specifier` is a `type-specifier`
    * `struct identifier_opt { struct-declaration-list }`
    * `struct identifier`
  * `struct-declaration-list:`
    * `struct-declaration`
    * `struct-declaration struct-declaration-list`
  * `struct-declaration:`
    * `specifier-qualifier-list struct-declarator-list ;`
* A declaration must have at least one declarator, or its type specifier must
  declare a `struct`.
