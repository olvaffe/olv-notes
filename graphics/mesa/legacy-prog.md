Mesa Vertex and Fragment Programs
=================================

## Example

- Snippet

    const GLchar src[] = "
    !!ARBfp1.0
    TEMP R0;
    PARAM c[4] = { { 0, 0, 0, 0 },
                   program.local[0..1],
                   { 1, 1, 1, 1 } };
    MOV R0, c[1];
    SUB R0, R0, c[0];
    ADD R0, R0, c[2];
    MUL R0, R0, c[3];
    MOV result.color, R0;
    END";

    glGenProgramsARB(1, &prog);
    glBindProgramARB(GL_FRAGMENT_PROGRAM_ARB, prog);
    glProgramStringARB(GL_FRAGMENT_PROGRAM_ARB, GL_PROGRAM_FORMAT_ASCII_ARB,
                    size, src);
    glEnable(GL_FRAGMENT_PROGRAM_ARB);
    glProgramLocalParameter4fARB(GL_FRAGMENT_PROGRAM_ARB, 0,
                                 1.0, 1.0, 0.0, 0.0);
    glProgramLocalParameter4fARB(GL_FRAGMENT_PROGRAM_ARB, 1,
                                 0.0, 0.0, 1.0, 1.0);
    glRectf(x1, y1, x2, y2);

- `glGenProgramARB` creates a `struct gl_program` lazily
  - created with `st_new_program`
- `glBindProgramARB` makes a program current
  - it sets `_NEW_PROGRAM | _NEW_PROGRAM_CONSTANTS`
  - calls `st_bind_program` to notify the driver
- `glProgramStringARB` compiles a program
  - it sets `_NEW_PROGRAM`
  - calls `_mesa_parse_arb_fragment_program`
  - `st_program_string_notify`
- `glEnable(GL_FRAGMENT_PROGRAM_ARB)` enables fp
  - it sets `_NEW_PROGRAM`
- `glProgramLocalParameter4fARB` sets local parameters
  - it sets `_NEW_PROGRAM_CONSTANTS`
  - no st call
- `_mesa_update_state` is called before drawing
  - `update_program_enables` really enables programs if they are enabled, and
    are valid.
  - `_mesa_update_texture` depends on the program because it has to enable
    textures used by a program
  - `update_program` sets the real current programs and call `st_bind_program`
    again
  - `update_arrays` depends on the vertex program because it has to know the
    attributes used by a vertex program.
  - `update_program_constants` passes `_NEW_PROGRAM_CONSTANTS` to the driver if
    there really are new constants
  - `st_invalidate_state`
- `st_validate_state` is called before drawing
  - `update_raster_state` enables two-sided lighting if the vertex program does,
    `sprite_coord_enable`, and `point_size_per_vertex`
  - `st_upload_constants` uploads every parameter (fixed-function states,
    program parameters, uniforms) to the constant buffer
  - `finalize_textures` and `update_textures` for each sampler used
  - `update_fp` to translate MESA IR to TGSI and bind to the pipe driver

## `ARB_vertex_program`

- is not in the OpenGL core.
- is enabled by `glEnable(GL_VERTEX_PROGRAM_ARB);`
  - `glUseProgramObjectARB` takes precedence over vertex program
- defines `VertexAttrib*` functions as `ARB_vertex_shader` does.
- defines program objects, but different from shader program objects
  - created by `glGenProgramsARB`
  - deleted by `glDeleteProgramARB`
  - made active by `glBindProgramARB`
  - source code is specified by `glProgramStringARB`
  - queryed by `glGetProgramivARB`
    - it is different from `glGetProgramiv`!!
- each program has an array of
  - local parameters, specified by `glProgramLocalParameter*`
  - environment parameters, specified by `glProgramEnvParameter*`
    - a environment parameter is shared by all programs
- Program String
  - attributes are accessed through `vertex.<attr>`
    - marked in `gl_program::InputsRead`: `VERT_BIT_<...>`
  - states such as material or lighting are accessed through `state.material` or `state.lightmodel`
    - `gl_program::Parameters`
  - parameters are accessed through `program.env` or `program.local`
    - local `gl_program::Parameters`
    - env `gl_vertex_program_state::Parameters`
  - outputs are defined through `result.<attr>`
    - marked in `gl_program::OutputsWritten`: `VERT_RESULT_<...>`
  - addresses: for indirect access
  - temporaries

## `ARB_fragment_program`

- See `ARB_vertex_program` above
- `gl_program::InputsRead` has `FRAG_BIT_<...>`
- `gl_program::OutputsWritten` has `FRAG_RESULT_<...>`

## Translate ARBfp to MESA IR

- `_mesa_parse_arb_fragment_program` translates the example snippet into

    # Fragment Program/Shader 1
    0: MOV TEMP[0], STATE[0];
    1: SUB TEMP[0], TEMP[0], CONST[1];
    2: ADD TEMP[0], TEMP[0], STATE[2];
    3: MUL TEMP[0], TEMP[0], CONST[3];
    4: MOV OUTPUT[2], TEMP[0];
    5: END

- the parser is written in bison, `program_parse.y`
- MESA IR is an editable representation

## Translate MESA IR to TGSI

- `st_translate_fragment_program` translates the above MESA IR into

    FRAG
    PROPERTY FS_COLOR0_WRITES_ALL_CBUFS 1
    DCL OUT[0], COLOR
    DCL CONST[0]
    DCL CONST[2]
    DCL TEMP[0]
    IMM FLT32 {    0.0000,     1.0000,     0.0000,     0.0000}
    0: MOV TEMP[0], CONST[0]
    1: SUB TEMP[0], TEMP[0], IMM[0].xxxx
    2: ADD TEMP[0], TEMP[0], CONST[2]
    3: MUL TEMP[0], TEMP[0], IMM[0].yyyy
    4: MOV OUT[0], TEMP[0]
    5: END

- `_mesa_remove_output_reads` to replace output reading by temp reading
