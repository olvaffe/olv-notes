NIR
===

## Variables

- a `nir_variable` represents a variable in a high-level language
  - it encapsulates all info about the high-level variable, such as type,
    qualifiers, dimensions, etc.
  - but it is not directly usable
- a `nir_deref_instr` results in a "pointer" to the variable
  - `dest` is a SSA `nir_dest`
  - `dest.ssa.parent_instr->var` points back to the variable
  - the dest is a pointer and is still not directly usable
- a `nir_instrinsic_instr` with op `nir_intrinsic_load_deref` loads the value
  - `instr->src[0].ssa` points to the "pointer"
  - `instr->dest.ssa` holds the variable value
- when the variable is more than a vec4, such as a matrix or an array or a
  struct, multiple `nir_deref_instr` are needed to get a `nir_dest` that can
  be used as a `nir_src` in `nir_intrinsic_load_deref`

## Instructions

- use `nir_alu_instr` as an example
  - created with `nir_alu_instr_create`
  - `op` specifies the alu op such as `nir_op_mov` or `nir_op_fadd`
  - real size depends on the number of srcs of `op`
  - `exact` means transforms must result in bit-exact result
  - a `nir_alu_dest` is a `nir_dest` plus
    - `saturate`
    - `write_mask`
  - a `nir_alu_src` is a `nir_src` plus
    - `negate`
    - `abs`
    - `swizzle`
- a `nir_instr` works with `nir_dest` and `nir_src`
  - a `nir_dest` is either a `nir_reg_dest` or a `nir_ssa_def`, depending on
    whether it is in SSA form or not
  - a `nir_src` is either a `nir_reg_src` or a `nir_ssa_def`,  depending on
    whether it is in SSA form or not
  - after converting out of SSA form, we use `nir_reg_dest` or `nir_reg_src`,
    which has a pointer to a `nir_register` to represent a physical register
- a `nir_variable` should be derefed and loaded using special `nir_instr`s to
  create a `nir_dest`, before it is usable by normal `nir_instr`

## Control Flows

- A `nir_function` represents a high-level function declaration.  It is not a
  `nir_cf_node`.  But it can have a `nir_function_impl` which is a
  `nir_cf_node`
- a `nir_cf_node` tree consists of
  - a `nir_function_impl` which is the root node
  - `nir_loop` and `nir_if` which are branch nodes
  - `nir_block` which are leaf nodes
  - the first and the last children of a root node or a branch node always
    exist and are always `nir_block`
    - `nir_function_impl_create_bare` adds a `nir_block` child
    - `nir_loop_create` adds a `nir_block` child
    - `nir_if_create` adds two `nir_block` children, for "then" and "else"
      respectively
    - a sibling node is added with `nir_cf_node_insert`, which always
      surrounds the node with `nir_block`
  - to go to the root node, `nir_cf_node_get_function`
  - to move between sibling nodes,
    - `nir_cf_node_next`
    - `nir_cf_node_prev`
    - `nir_cf_node_is_first`
    - `nir_cf_node_is_last`
- `nir_function_impl`
  - `body` is the list of the children nodes
  - `nir_function_impl_create_bare` always adds a start block as the first
    child
  - there is also a special end block that is not added to the tree
  - `nir_start_block` returns the start block
  - `nir_impl_last_block` returns the last block
    - which is not the special end block
- `nir_loop`
  - `body` is a list of the children nodes
  - `nir_loop_create` always adds a block as the first child
- `nir_if`
  - `then_list` and `else_list` are two lists of the children nodes
  - `nir_if_create` always adds a block to each list as the first children
- `nir_block`
  - `instr_list` is a list of instructions
  - all `nir_block` nodes in the tree also form a control flow graph, CFG
  - `predecessors` and `successors` are edges of the CFG

## SPIR-V to NIR

- GLSL
  - `in vec4 in_color;`
  - `out vec4 out_color;`
  - `void main() { out_color = in_color; }`
- `spirv_to_nir`
- NIR
  - `decl_var shader_in vec4 in_color (VARYING_SLOT_VAR0.xyzw, 0, 0)`
  - `decl_var shader_out vec4 out_color (FRAG_RESULT_DATA0.xyzw, 0, 0)`
  - `vec1 ssa0 = deref_var &in_color (shader_in vec4)`
  - `vec1 ssa1 = deref_var &out_color (shader_out vec4)`
  - `vec4 ssa2 = intrinsic load_deref(ssa0) (0/*access*/)`
  - `instrinsic store_deref(ssa1, ssa2) (15/*wrmask*/, 0/*access*/)`
- `NIR_PASS` is for normal passes
- `NIR_PASS_V` is for passes that do not return progress.  V stands for
  variant.
- compile
  - `nir_lower_io_to_temporaries` adds a temp var.  It replaces `store_deref`
    dest by a temp deref and insert a `copy_deref` from the temp deref to the
    real dest
  - `nir_lower_global_vars_to_local` moves the temp var into `main`
  - `nir_lower_vars_to_ssa` undoes `nir_lower_io_to_temporaries` but leaves
    some dead code
  - `nir_lower_alu_to_scalar` scalarizes a vec4 mov alu to 4 float mov alu
  - `nir_copy_prop` makes the mov alus dead code
  - `nir_opt_dce` removes the dead code
  - `nir_remove_dead_variables` removes the dead temp var
- the compiled NIR can be saved to disk cache
- link
  - `nir_lower_io_to_scalar_early` scalarizes `in_color`
  - `nir_link_opt_varyings` replaces `in_color.w` by 1.0
    - it knows VS writes 1.0 for W
  - `nir_lower_io_to_vector` undoes scalarization of `in_color`
     - it knows there is no need to load `in_color.w` now
- postprocess
  - lower inputs
    - for each input var,
      `var->data.driver_location = var->data.location`, where
      `location` is `VARYING_SLOT_VAR0`
    - `var->data.interpolation = INTERP_MODE_SMOOTH` unless flat shading or
      explicitly set
    - `nir_lower_io(nir, nir_var_shader_in, type_size_vec4, ...)` finds the
      `load_deref` and replaces it by
      - `load_const(0.0)`
      - `intrinsic load_barycentric_pixel`
      - `intrinsic load_interpolated_input` w/ base set to
      	`var->data.driver_location`
  - lower outputs
    - for each output var,
      `var->data.driver_location = var->data.location`
    - `nir_lower_io(nir, nir_var_shader_out, type_size_vec4, ...)` finds the
      `store_deref` and replaces it by
       - `load_const(0.0)`
       - `intrinsic store_output` w/ base set to `var->data.driver_location`
  - `nir_opt_dce` removes dead code
  - `nir_opt_cse` removes common subexpressions
    - e.g., removes all but one `load_const(0.0)`
- final NIR
  - `vec2 ssa0 = intrinsic load_barycentric_pixel () (/*interp_mode*/1)`
  - `vec1 ssa1 = load_const (0.0f)`
  - `vec3 ssa2 = intrinsic load_interpolated_input (ssa0, ssa1) (/*base*/32, /*component*/0)`
  - `vec1 ssa3 = load_const (1.0f)`
  - `vec4 ssa4 = vec4 ssa2.x, ssa2.y, ssa2.z, ssa3`
  - `intrinsic store_output (ssa4, ssa1) (/*base*/8, /*wrmast*/15, /*component*/0, /*type*/160)`
- codegen

## GLSL to NIR

- tools
  - `glsl_compiler` to compile GLSL to GLSL IR
  - `spirv2nir` to compile SPIR-V to NIR
  - `MESA_GLSL=dump` to `_mesa_print_ir` GLSL IR and `nir_print_shader` NIR
- types
  - `struct gl_shader`, created by glCreateShader
  - `struct gl_shader_program`, created by `glCreateProgram`
  - `struct gl_linked_shader`, inside `struct gl_shader_program` and created
    by `glLinkProgram`
  - `struct gl_program`, inside `struct gl_linked_shader`
- each GLSL snippet is compiled to GLSL IR using `_mesa_glsl_compile_shader`
  - `_mesa_glsl_parse` is called to parse GLSL into AST
  - `_mesa_ast_to_hir` is called to parse AST into HIR, with validations
  - fundamental passes are applied to gather info and lower HIR to LIR (both
    are GLSL IR)
  - all artifacts are saved to `struct gl_shader`
- `struct glsl_shader` are collected into a `struct gl_shader_program` and
  linked using `_mesa_glsl_link_shader`
  - it calls common `link_shaders` to cross validate and optimize GLSL IRs
  - it calls backend's `LinkShader` function
    - in st case, `st_link_shader`
    - `st_link_shader` applies more lowering and optimization passes
    - it then calls `st_link_nir` to `st_glsl_to_nir` GLSL IR to NIR for each
      `struct gl_program`
- at draw time, when all states are known, a variant of a `struct gl_program`
  is created and cached
  - in st case, it first clones NIR, calls `st_finalize_nir`, and call into
    the gallium driver to get the CSO.  It then binds the CSO and call
    `draw_vbo`
  - gallium drivers only see the finalized NIR
- example
  - `glCreateShader`:
        varying vec4 vFillColor;
        void main() {
          gl_FragColor = vFillColor;
        }
  - `glCompileShader`:
        (
        (declare (location=2 shader_out ) vec4 gl_FragColor)
        (declare (location=25 shader_in ) vec2 gl_PointCoord)
        (declare (location=24 shader_in ) bool gl_FrontFacing)
        (declare (location=0 shader_in ) vec4 gl_FragCoord)
        (declare (location=2 shader_in ) vec4 gl_SecondaryColor)
        (declare (location=1 shader_in ) vec4 gl_Color)
        (declare (location=3 shader_in ) float gl_FogFragCoord)
        (declare (location=4 shader_in ) (array vec4 0) gl_TexCoord)
        (declare (uniform ) (array mat4 8) gl_TextureMatrixInverseTranspose)
        (declare (uniform ) (array mat4 8) gl_TextureMatrixTranspose)
        (declare (uniform ) mat4 gl_ModelViewProjectionMatrixInverseTranspose)
        (declare (uniform ) mat4 gl_ProjectionMatrixInverseTranspose)
        (declare (uniform ) mat4 gl_ModelViewMatrixInverseTranspose)
        (declare (uniform ) mat4 gl_ModelViewProjectionMatrixTranspose)
        (declare (uniform ) mat4 gl_ProjectionMatrixTranspose)
        (declare (uniform ) mat4 gl_ModelViewMatrixTranspose)
        (declare (uniform ) mat4 gl_ModelViewProjectionMatrix)
        (declare (shader_in ) vec4 vFillColor)
        ( function main
         (signature void (parameters)
           (
             (assign  (xyzw) (var_ref gl_FragColor)  (var_ref vFillColor) )
           ))
        )
        )
    - it implicitly declares of built-in variables and uniforms
      - many are outdated and are not used by modern shaders
    - `vFillColor` has location -1 because no explicit location is given
  - `glLinkProgram`:
        (
        (declare (location=31 shader_in ) vec4 vFillColor)
        (declare (location=2 shader_out ) vec4 gl_FragColor)
        (declare (temporary ) vec4 gl_FragColor)
        ( function main
          (signature void (parameters)
            (
              (assign  (xyzw) (var_ref gl_FragColor)  (var_ref vFillColor) )
              (assign  (xyzw) (var_ref gl_FragColor@4)  (var_ref gl_FragColor) )
            ))
        )
        )
    - dead variables are eliminated
    - locations are assigned to `vFillColor`
    - NIR is also created
          shader: MESA_SHADER_FRAGMENT
          name: GLSL1
          inputs: 0
          outputs: 0
          uniforms: 0
          shared: 0
          decl_var shader_in INTERP_MODE_NONE float vFillColor (VARYING_SLOT_VAR0.x, 0, 0)
          decl_var shader_in INTERP_MODE_NONE float vFillColor@0 (VARYING_SLOT_VAR0.y, 0, 0)
          decl_var shader_in INTERP_MODE_NONE float vFillColor@1 (VARYING_SLOT_VAR0.z, 0, 0)
          decl_var shader_in INTERP_MODE_NONE float vFillColor@2 (VARYING_SLOT_VAR0.w, 0, 0)
          decl_var shader_out INTERP_MODE_NONE vec4 gl_FragColor (FRAG_RESULT_COLOR, 0, 0)
          decl_function main (0 params)
          
          impl main {
                  decl_var  INTERP_MODE_NONE vec4 out@gl_FragColor-temp
                  decl_var  INTERP_MODE_NONE vec4 in@vFillColor-temp
                  decl_var  INTERP_MODE_NONE vec4 gl_FragColor@3
                  block block_0:
                  /* preds: */
                  vec1 32 ssa_48 = deref_var &vFillColor (shader_in float) 
                  vec1 32 ssa_49 = intrinsic load_deref (ssa_48) (0) /* access=0 */
                  vec1 32 ssa_50 = deref_var &vFillColor@0 (shader_in float) 
                  vec1 32 ssa_51 = intrinsic load_deref (ssa_50) (0) /* access=0 */
                  vec1 32 ssa_52 = deref_var &vFillColor@1 (shader_in float) 
                  vec1 32 ssa_53 = intrinsic load_deref (ssa_52) (0) /* access=0 */
                  vec1 32 ssa_54 = deref_var &vFillColor@2 (shader_in float) 
                  vec1 32 ssa_55 = intrinsic load_deref (ssa_54) (0) /* access=0 */
                  vec4 32 ssa_56 = vec4 ssa_49, ssa_51, ssa_53, ssa_55
                  vec1 32 ssa_6 = deref_var &gl_FragColor (shader_out vec4) 
                  intrinsic store_deref (ssa_6, ssa_56) (15, 0) /* wrmask=xyzw */ /* access=0 */
                  /* succs: block_1 */
                  block block_1:
          }

## NIR to backend

- after translation from GLSL/SPIR-V and a pass to SSA, NIR still uses
  variables (`deref_var`, `load_deref`, and `store_deref`) heavily
- to lower variables, one needs to lower derefs to appropricate intrinsics
  - "system value" variable derefs to `nir_intrinsic_load_<sv>`
  - "in" variable derefs to `nir_intrinsic_load_input` or others
  - "out" variable derefs to `nir_intrinsic_store_output` or others
  - "uniform" variable derefs to `nir_intrinsic_load_uniform`
- however, those intrinsics require `driver_location` to be set
  - and?
- freedreno tools
  - `ir3_compiler` to compile GLSL to IR3
  - `IR3_SHADER_DEBUG=disasm` to `nir_print_shader` final NIR and
    `ir3_shader_disasm` binary
- freedreno
  - in CSO creation, `ir3_shader_from_nir` is called to `nir_lower_io` and
    `ir3_optimize_nir` final NIR
  - in `draw_vbo`, `ir3_shader_get_variant` is called to
    `ir3_compile_shader_nir` NIR to IR3 and `ir3_shader_assemble` IR3

## IO

- examples
  - loading `in vec4 in_val` is translated to a `deref_var` and a `load_deref`
  - loading `in centroid vec4 in_val` or `interpolateAtCentroid` is translated
    to a `deref_var` and a `interp_deref_at_centroid`
- `nir_lower_io`
  - `load_deref` is lowered to `load_barycentric_pixel` and
    `load_interpolated_input`
  - `interp_deref_at_centroid` is lowered to `load_barycentric_centroid` and
    `load_interpolated_input`
